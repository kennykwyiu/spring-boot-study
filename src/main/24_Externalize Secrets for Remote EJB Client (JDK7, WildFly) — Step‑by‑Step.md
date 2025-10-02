# Externalize Secrets for Remote EJB Client (JDK7, WildFly) — Step‑by‑Step

> This guide shows how to externalize secrets and environment‑specific values for a remote EJB client packaged as a JAR (JDK 7), used as a dependency, connecting to WildFly. It keeps secrets out of source control and outside the JAR, and works on bare VMs or app servers.
> 

---

### Overview

- Objective: remove secrets and environment-specific constants from code and JAR
- Target: remote EJB client used by another app, compiled with JDK 7, deployed against WildFly
- Approach: replace hardcoded finals with runtime lookups from an external properties file and external [jboss-ejb-client.properties](http://jboss-ejb-client.properties)
- Security: file permissions, optional Elytron credential store on WildFly

---

### 1) Inventory and classify your constants

- Scan the constants class and list fields.
- Classify:
    - Secrets: passwords, API keys, tokens
    - Environment values: hosts, ports, JNDI names, feature flags
    - True constants: compile-time constants that are not environment-specific and not sensitive
- Goal:
    - Secrets and environment values → move to external config
    - True constants → can stay as code

---

### 2) Create an external properties file

- Create a secure file outside classpath and JAR, for example:
    - Linux: `/etc/newapp/[newapp.properties](http://newapp.properties)`
- Suggested keys:

```
api.key=REDACTED
api.secret=REDACTED
[remote.host](http://remote.host)=[ejb.example.com](http://ejb.example.com)
remote.port=8080
remote.protocol=http-remoting
ejb.remote.user=service-user
ejb.remote.password=REDACTED
[jndi.name](http://jndi.name)=ejb:/app/module//BeanName!com.example.MyRemote
```

- Permissions:
    - Owner-only read: `chmod 600 /etc/newapp/[newapp.properties](http://newapp.properties)`
    - Ownership: set to the service user running the JVM, for example: `chown appuser:app /etc/newapp/[newapp.properties](http://newapp.properties)`

---

### 3) Load config in JDK 7 safely

Create a tiny static loader that reads from an absolute path provided via a system property.

```java
package com.example.config;

import [java.io](http://java.io).FileInputStream;
import [java.io](http://java.io).IOException;
import [java.util.Properties](http://java.util.Properties);

public final class AppConfig {
	private static final Properties P = new Properties();
	static {
		final String path = System.getProperty("newapp.config.path");
		if (path == null) {
			throw new IllegalStateException("Missing -Dnewapp.config.path");
		}
		FileInputStream in = null;
		try {
			in = new FileInputStream(path);
			P.load(in);
		} catch (IOException e) {
			throw new RuntimeException("Cannot load config " + path, e);
		} finally {
			if (in != null) try { in.close(); } catch (IOException ignore) {}
		}
	}

	public static String get(String key) {
		final String v = P.getProperty(key);
		if (v == null || v.isEmpty()) {
			throw new IllegalStateException("Missing config: " + key);
		}
		return v;
	}

	public static int getInt(String key) {
		return Integer.parseInt(get(key));
	}
}
```

- Launch your JVM with:

```bash
-Dnewapp.config.path=/etc/newapp/[newapp.properties](http://newapp.properties)
```

---

### 4) Refactor the constants class

- Replace hardcoded finals that vary by environment with values read from AppConfig.
- If you must keep fields constant after startup, you can still keep them `final` and assign from `AppConfig` at initialization time.

Before:

```java
public final class Constants {
	public static final String API_KEY = "hardcoded";
	public static final String REMOTE_HOST = "[localhost](http://localhost)";
	public static final String JNDI_NAME = "ejb:/old";
}
```

After:

```java
import static com.example.config.AppConfig.get;

public final class Constants {
	public static final String API_KEY = get("api.key");
	public static final String REMOTE_HOST = get("[remote.host](http://remote.host)");
	public static final String JNDI_NAME = get("[jndi.name](http://jndi.name)");
}
```

- For secrets used by EJB client auth, prefer not to store them as static finals in code. Retrieve them where needed or inject via constructor and cache in memory.

---

### 5) Externalize [jboss-ejb-client.properties](http://jboss-ejb-client.properties)

You can use the standard client properties file, but keep it outside the JAR and reference it with a system property.

- Create `/etc/newapp/[jboss-ejb-client.properties](http://jboss-ejb-client.properties)`:

```
remote.connections=default

[remote.connection.default.host](http://remote.connection.default.host)=${[remote.host](http://remote.host)}
remote.connection.default.port=${remote.port}
remote.connection.default.protocol=${remote.protocol}
[remote.connection.default.connect.options.org](http://remote.connection.default.connect.options.org).xnio.Options.SASL_POLICY_NOANONYMOUS=false

remote.connection.default.username=${ejb.remote.user}
remote.connection.default.password=${ejb.remote.password}
```

- Some client versions do NOT support `${...}` substitution. If substitution is not supported, see step 6 for a programmatic setup.
- Point the JVM to the file:

```bash
-[Dorg.jboss.ejb.client.properties](http://Dorg.jboss.ejb.client.properties).file.path=/etc/newapp/[jboss-ejb-client.properties](http://jboss-ejb-client.properties)
```

---

### 6) Programmatic EJB client (fallback if substitution unsupported)

If property substitution isn’t supported by your client, construct the EJB context via code in JDK 7.

```java
import javax.naming.Context;
import javax.naming.InitialContext;
import [java.util.Properties](http://java.util.Properties);

public class EjbClientFactory {
	public static Context createContext() throws Exception {
		Properties p = new Properties();
		p.put("remote.connections", "default");
		p.put("[remote.connection.default.host](http://remote.connection.default.host)", AppConfig.get("[remote.host](http://remote.host)"));
		p.put("remote.connection.default.port", AppConfig.get("remote.port"));
		p.put("remote.connection.default.protocol", AppConfig.get("remote.protocol"));
		p.put("remote.connection.default.username", AppConfig.get("ejb.remote.user"));
		p.put("remote.connection.default.password", AppConfig.get("ejb.remote.password"));
		p.put("[remote.connection.default.connect.options.org](http://remote.connection.default.connect.options.org).xnio.Options.SASL_POLICY_NOANONYMOUS", "false");
		return new InitialContext(p);
	}
}
```

Usage:

```java
Context ctx = EjbClientFactory.createContext();
MyRemote proxy = (MyRemote) ctx.lookup(AppConfig.get("[jndi.name](http://jndi.name)"));
```

---

### 7) Wire into the dependent application

- Ensure the application that uses this EJB client JAR sets both system properties at startup:

```bash
-Dnewapp.config.path=/etc/newapp/[newapp.properties](http://newapp.properties) \
-[Dorg.jboss.ejb.client.properties](http://Dorg.jboss.ejb.client.properties).file.path=/etc/newapp/[jboss-ejb-client.properties](http://jboss-ejb-client.properties)
```

- If the dependent app is a WAR on WildFly:
    - You can still supply system properties via the server launch script, or
    - Use WildFly system properties in `standalone.conf` or via CLI, then read them within the app to determine file paths.

---

### 8) Secure handling on WildFly with Elytron (optional but recommended)

If the EJB client runs inside WildFly or the server needs secrets:

- Create an Elytron credential store and store secrets by alias.
- Reference aliases in datasources, realms, or custom code via Elytron APIs.
- Benefit: secrets are encrypted at rest and not stored in plaintext properties.

High-level steps:

1. Create credential store with Elytron tool or WildFly CLI.
2. Configure the Elytron subsystem in `standalone.xml` to reference the store.
3. Replace plaintext password usage with Elytron references.

---

### 9) Testing checklist

- Startup fails when the external files are missing or unreadable
    - Expectation: clear error on missing `-Dnewapp.config.path`
- EJB connectivity
    - Verify JNDI lookup succeeds using values from the external files only
- Secrets audit
    - Confirm JAR contains no secrets
    - Confirm repo contains no secrets
- Permissions
    - `ls -l` shows correct ownership and 600 perms on both files
- Logging hygiene
    - Ensure logs do not print secrets or full connection strings

---

### 10) Rollout plan

- Dev: create files locally and confirm connectivity
- Stage: deploy files under `/etc/newapp/`, set JVM properties, validate
- Prod: provision files via secure channel, set ownership and perms, enable JVM flags, validate
- Incident fallback: keep a sample [`newapp.properties](http://newapp.properties).example` under version control with placeholder values, never real secrets

---

### 11) Anti-patterns to avoid

- Storing secrets as classpath resources inside the JAR
- Keeping environment-dependent `public static final` literals in code
- Printing loaded config values that include secrets in logs
- Using WildFly Vault legacy instead of Elytron for new setups unless constrained

---

### 12) Minimal working example summary

- Files:
    - `/etc/newapp/[newapp.properties](http://newapp.properties)`
    - `/etc/newapp/[jboss-ejb-client.properties](http://jboss-ejb-client.properties)` (or programmatic config)
- JVM flags:

```bash
-Dnewapp.config.path=/etc/newapp/[newapp.properties](http://newapp.properties) \
-[Dorg.jboss.ejb.client.properties](http://Dorg.jboss.ejb.client.properties).file.path=/etc/newapp/[jboss-ejb-client.properties](http://jboss-ejb-client.properties)
```

- Code:
    - `AppConfig` loader (JDK 7 compatible)
    - Constants refactored to read from `AppConfig`
    - Optional programmatic EJB client factory

---

### 13) Next steps for your codebase

- Paste your current constants class and WildFly version
- I will generate a patched version that compiles on JDK 7 and replaces the finals appropriately
- I can also produce a server launch snippet specific to your environment (systemd, shell, or domain mode)