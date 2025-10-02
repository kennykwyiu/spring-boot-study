# Remote EJB — Use External jboss-ejb-client.properties (Step-by-step)

### Goal

Call a remote EJB using an external [jboss-ejb-client.properties](http://jboss-ejb-client.properties) file, not the one inside the WAR.

---

### Step 1 — Create the external EJB client properties

Create a file (UTF‑8) outside your WAR. Example: /etc/yourapp/[jboss-ejb-client.properties](http://jboss-ejb-client.properties)

Minimal content:

```
org.jboss.ejb.client.scoped.context=true
remote.connections=default
[remote.connection.default.host](http://remote.connection.default.host)=${EJB_HOST:[localhost](http://localhost)}
remote.connection.default.port=${EJB_PORT:8080}
remote.connection.default.username=${EJB_USERNAME}
remote.connection.default.password=${EJB_PASSWORD}
[remote.connectionprovider.create.options.org](http://remote.connectionprovider.create.options.org).xnio.Options.SSL_ENABLED=false
```

Tips:

- Use environment substitution as above so the same file works in all environments.
- If you use TLS, set SSL_ENABLED=true and configure truststore via wildfly-config.xml or additional props.

---

### Step 2 — Place and secure the file

- Linux example:
    - sudo mkdir -p /etc/yourapp
    - sudo chown appuser:appgroup /etc/yourapp/[jboss-ejb-client.properties](http://jboss-ejb-client.properties)
    - sudo chmod 640 /etc/yourapp/[jboss-ejb-client.properties](http://jboss-ejb-client.properties)

---

### Step 3 — Point the application to the external file

Pick ONE of these ways.

Option A — System property (recommended)

- Add to your start command or JAVA_OPTS:
    
    -[Djboss.ejb.client.properties](http://Djboss.ejb.client.properties).path=/etc/yourapp/[jboss-ejb-client.properties](http://jboss-ejb-client.properties)
    

Option B — Environment variable

- Linux: export JBOSS_EJB_CLIENT_PROPERTIES_PATH=/etc/yourapp/[jboss-ejb-client.properties](http://jboss-ejb-client.properties)
- Windows (PowerShell): $env:JBOSS_EJB_CLIENT_PROPERTIES_PATH="C:\yourapp\[jboss-ejb-client.properties](http://jboss-ejb-client.properties)"

Option C — Programmatic load (if you prefer code control)

- Load the file yourself and pass it to InitialContext (see Appendix).

---

### Step 4 — If you also externalize Spring configs

Pass multiple locations as a comma‑separated list.

Examples:

- Linux/macOS:
    
    -Dspring.config.additional-location=optional:file:/etc/yourapp/,optional:file:/run/secrets/[app.properties](http://app.properties)
    
- Windows (PowerShell):
    
    "-Dspring.config.additional-location=optional:file:/C:/yourapp/,optional:file:/C:/secrets/[app.properties](http://app.properties)"
    

---

### Step 5 — Update your start script (bash example)

```bash
# [setenv.sh](http://setenv.sh) or [start.sh](http://start.sh)
EXTRA_CFG="optional:[file:/etc/yourapp/,optional:file:/run/secrets/app.properties](file:/etc/yourapp/,optional:file:/run/secrets/app.properties)"
EJB_PROPS="/etc/yourapp/[jboss-ejb-client.properties](http://jboss-ejb-client.properties)"

JAVA_OPTS+=" -Dspring.config.additional-location=${EXTRA_CFG}"
JAVA_OPTS+=" -[Djboss.ejb.client.properties](http://Djboss.ejb.client.properties).path=${EJB_PROPS}"

# Optional (timeouts, TZ, memory)
# JAVA_OPTS+=" -Dorg.jboss.ejb.client.connect.timeout=10000 -Dorg.jboss.ejb.client.invocation.timeout=20000"
# JAVA_OPTS+=" -Duser.timezone=Asia/Hong_Kong -Xms512m -Xmx1024m"

export JAVA_OPTS
exec java ${JAVA_OPTS} -jar app.jar
```

Tomcat users:

- Put the flags in CATALINA_OPTS in CATALINA_BASE/bin/[setenv.sh](http://setenv.sh)

Systemd users:

- Environment="JAVA_OPTS= -Dspring.config.additional-location=... -[Djboss.ejb.client.properties](http://Djboss.ejb.client.properties).path=/etc/yourapp/[jboss-ejb-client.properties](http://jboss-ejb-client.properties)"

---

### Step 6 — InitialContext creation patterns

A) Rely on external properties (simplest)

```java
InitialContext ctx = new InitialContext();
```

B) Add URL_PKG_PREFIXES (only if needed for URL-style lookups)

```java
Properties env = new Properties();
env.put(Context.INITIAL_CONTEXT_FACTORY, "org.wildfly.naming.client.WildFlyInitialContextFactory");
env.put(Context.URL_PKG_PREFIXES, "org.jboss.ejb.client.naming");
InitialContext ctx = new InitialContext(env);
```

C) Programmatic external load (full control)

```java
Path p = Paths.get(System.getProperty("[jboss.ejb.client.properties](http://jboss.ejb.client.properties).path",
           System.getenv("JBOSS_EJB_CLIENT_PROPERTIES_PATH")));
Properties env = new Properties();
try (BufferedReader r = Files.newBufferedReader(p.toAbsolutePath().normalize(), StandardCharsets.UTF_8)) {
    env.load(r);
}
env.putIfAbsent(Context.INITIAL_CONTEXT_FACTORY, "org.wildfly.naming.client.WildFlyInitialContextFactory");
env.putIfAbsent("jboss.naming.client.ejb.context", "true");
InitialContext ctx = new InitialContext(env);
```

Modern alternative (Elytron):

- Keep all client config in wildfly-config.xml outside the WAR and run with:
    
    -Dwildfly.config.url=[file:/etc/yourapp/wildfly-config.xml](file:/etc/yourapp/wildfly-config.xml)
    
- Then simply:

```java
InitialContext ctx = new InitialContext();
```

---

### Step 7 — Verify the JNDI lookup name

Use the EJB client API name:

- ejb:/<appName or empty>/<moduleName>/<beanName>!<fully.qualified.RemoteInterface>

Notes:

- WAR only: appName = ""; moduleName = your WAR name (without extension)
- EAR: set appName explicitly if needed
- distinctName: usually empty unless configured

---

### Step 8 — Smoke test

- Export envs for credentials:
    - export EJB_HOST=[server.example.com](http://server.example.com)
    - export EJB_PORT=8080
    - export EJB_USERNAME=appuser
    - export EJB_PASSWORD=
- Start the app and invoke a simple remote method.
- Increase client logging temporarily for: org.jboss.ejb.client, org.xnio, org.wildfly.naming.client

---

### Step 9 — Troubleshooting “No EJB receiver available”

- Confirm the external properties file path you set is actually read; log the absolute path on startup (avoid logging secrets).
- Ensure http-remoting is enabled on the server and the port matches your client.
- Verify the bean is deployed with @Remote and your ejb:/ name matches app/module/bean/interface.
- Confirm correct client dependencies (WildFly EJB client) and remove conflicting legacy remoting libs.
- If behind a reverse proxy, make sure HTTP upgrade is allowed for remoting.

---

### Appendix — Minimal helper to load external props

```java
public final class EjbContexts {
  private static Path resolveExternalPath() {
    String sys = System.getProperty("[jboss.ejb.client.properties](http://jboss.ejb.client.properties).path");
    if (sys != null && !sys.isBlank()) return Paths.get(sys);
    String env = System.getenv("JBOSS_EJB_CLIENT_PROPERTIES_PATH");
    if (env != null && !env.isBlank()) return Paths.get(env);
    return Paths.get("[jboss-ejb-client.properties](http://jboss-ejb-client.properties)"); // working dir fallback
  }
  public static InitialContext fromExternalProps() throws NamingException, IOException {
    Path path = resolveExternalPath().toAbsolutePath().normalize();
    if (!Files.isReadable(path)) throw new IllegalStateException("EJB client properties not readable: " + path);
    Properties p = new Properties();
    try (BufferedReader r = Files.newBufferedReader(path, StandardCharsets.UTF_8)) { p.load(r); }
    p.putIfAbsent(Context.INITIAL_CONTEXT_FACTORY, "org.wildfly.naming.client.WildFlyInitialContextFactory");
    p.putIfAbsent("jboss.naming.client.ejb.context", "true");
    return new InitialContext(p);
  }
}
```