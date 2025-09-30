# Externalize secrets — implementation plan

### Goal

- Externalize all hard-coded secrets so they are not inside the WAR

### 1) Decide your new app name

- Replace every occurrence of the string "newapp" in package names, URLs, paths, and UI routes
- Examples to search for:
    - Package and imports: com.hkecic.newapp...
    - Servlet paths and Vaadin Navigator routes: /newapp/... (e.g., /newapp/mainui, /newapp/proposalui, /newapp/fcui)
    - Static text in UI labels or titles
    - Build scripts and server configs (Nginx, Tomcat context paths)

---

### 2) Externalize sensitive information

Replace these former constants with external configuration (file, env vars, or JNDI):

- api.access.username
- api.access.password
- api.access.client_id
- api.access.client_secret

### Option A — External properties file (outside the WAR)

1) Create a file on the server, e.g. /etc/yourapp/[application-secrets.properties](http://application-secrets.properties)

```jsx
api.access.username=${API_ACCESS_USERNAME}
api.access.password=${API_ACCESS_PASSWORD}
api.access.client_id=${API_ACCESS_CLIENT_ID}
api.access.client_secret=${API_ACCESS_CLIENT_SECRET}
```

2) Provide the values via environment variables at deploy time.

3) Add a small loader class:

```java
public final class ApiAccessCfg {
    public static String USERNAME, PASSWORD, CLIENT_ID, CLIENT_SECRET;
    static {
        Properties p = new Properties();
        String path = System.getProperty("yourapp.secrets", "/etc/yourapp/");
        // load file, then assign fields
    }
}
```

4) Replace usages:

```java
// Before
// tokenRequest.username(API_ACCESS_TOKEN_USERNAME);

// After
tokenRequest.username(ApiAccessCfg.USERNAME);
```

5) Start the server with:

- -Dyourapp.secrets=/etc/yourapp/[application-secrets.properties](http://application-secrets.properties)

### Option B — Spring @Value with external file

```java
@Bean
public static 
```

```java
@Component
public class ApiAccessCfgBean {
    @org.springframework.beans.factory.annotation.Value("${api.access.username}") public String username;
    @org.springframework.beans.factory.annotation.Value("${api.access.password}") public String password;
    @org.springframework.beans.factory.annotation.Value("${api.access.client_id}") public String clientId;
    @org.springframework.beans.factory.annotation.Value("${api.access.client_secret}") public String clientSecret;
}
```

### Option C — JNDI (Tomcat context.xml)

```xml
<Environment name="api.access.client_secret" type="java.lang.String" value="${API_ACCESS_CLIENT_SECRET}"/>
```

```java
var env = (javax.naming.Context) new javax.naming.InitialContext().lookup("java:comp/env");
String secret = (String) env.lookup("api.access.client_secret");
```

---

### 3) Update URL and path references

- If you have report frameset base like InitVar.frameset_url, move it to an external property (e.g., yourapp.frameset_url) and update references in presenters and filters.
- Align reverse proxy paths (Nginx) and CORS allow-lists to new /newapp path.

---

### 4) Suggested find-and-replace checklist

- Code: "newapp" in packages, imports, and strings
- Web paths: /newapp
- Build: artifactId, WAR name, context path
- Config: property keys from eclink. *→ newapp.*
- CI/CD: update service names, secrets, and file paths under /etc/yourapp

---

### 5) Security hygiene

- Do not commit secrets to VCS
- Set file perms: chown appuser:appgroup /etc/yourapp; chmod 640 [application-secrets.properties](http://application-secrets.properties)
- Prefer a secret manager for the client_secret when possible

---

### 6) Fill-in section for you

- New app name: newapp
- New base URL path: /newapp
- Secrets delivery method: File | Env | JNDI | Vault
- External secrets file path: /etc/yourapp/[application-secrets.properties](http://application-secrets.properties)

/etc/yourapp/[application-secrets.properties](http://application-secrets.properties)

---

### Tomcat setup for newapp

### 1) Deploy at /newapp

- Create conf/Catalina/[localhost/newapp.xml](http://localhost/newapp.xml):

```xml
<Context docBase="/opt/apps/newapp.war" reloadable="false">
  <Parameter name="[java.io](http://java.io).tmpdir" value="/var/tmp/newapp" override="true"/>
</Context>
```

- Place the WAR at /opt/apps/newapp.war. Tomcat will serve it at /newapp.

### 2) External secrets via system property (file outside WAR)

- Create /etc/newapp/[application-secrets.properties](http://application-secrets.properties) with restricted perms:

```
api.access.username=...
api.access.password=...
api.access.client_id=...
api.access.client_secret=...
```

- TOMCAT_HOME/bin/[setenv.sh](http://setenv.sh):

```bash
#!/usr/bin/env bash
export CATALINA_OPTS="$CATALINA_OPTS -Dnewapp.secrets=/etc/newapp/[application-secrets.properties](http://application-secrets.properties)"
```

- Permissions:

```bash
sudo mkdir -p /etc/newapp
sudo chown tomcat:tomcat /etc/newapp /etc/newapp/[application-secrets.properties](http://application-secrets.properties)
sudo chmod 750 /etc/newapp
sudo chmod 640 /etc/newapp/[application-secrets.properties](http://application-secrets.properties)
```

- Java loader snippet (assign once at startup):

```java
public final class ApiAccessCfg {
  public static String USERNAME, PASSWORD, CLIENT_ID, CLIENT_SECRET;
  static {
    String path = System.getProperty("newapp.secrets", "/etc/newapp/[application-secrets.properties](http://application-secrets.properties)");
    var p = new [java.util.Properties](http://java.util.Properties)();
    try (var in = new [java.io](http://java.io).FileInputStream(path)) { p.load(in); }
    USERNAME = req(p, "api.access.username");
    PASSWORD = req(p, "api.access.password");
    CLIENT_ID = req(p, "api.access.client_id");
    CLIENT_SECRET = req(p, "api.access.client_secret");
  }
  private static String req([java.util.Properties](http://java.util.Properties) p, String k){
    var v = p.getProperty(k);
    if (v == null || v.isEmpty()) throw new IllegalStateException("Missing key: "+k);
    return v;
  }
}
```

### 3) External secrets via JNDI (alternative)

- conf/Catalina/[localhost/newapp.xml](http://localhost/newapp.xml):

```xml
<Context>
  <Environment name="api.access.username" type="java.lang.String" value="${API_ACCESS_USERNAME}"/>
  <Environment name="api.access.password" type="java.lang.String" value="${API_ACCESS_PASSWORD}"/>
  <Environment name="api.access.client_id" type="java.lang.String" value="${API_ACCESS_CLIENT_ID}"/>
  <Environment name="api.access.client_secret" type="java.lang.String" value="${API_ACCESS_CLIENT_SECRET}"/>
</Context>
```

- Provide variables via systemd override for Tomcat:

```
# sudo systemctl edit tomcat
[Service]
Environment="API_ACCESS_USERNAME=..."
Environment="API_ACCESS_PASSWORD=..."
Environment="API_ACCESS_CLIENT_ID=..."
Environment="API_ACCESS_CLIENT_SECRET=..."
```

- Lookup in code:

```java
var env = (javax.naming.Context) new javax.naming.InitialContext().lookup("java:comp/env");
String user = (String) env.lookup("api.access.username");
```

### 4) Nginx fronting Tomcat (optional)

```
location /newapp/ {
  proxy_pass [http://127.0.0.1:8080](http://127.0.0.1:8080);
  proxy_set_header Host $host;
  proxy_set_header X-Forwarded-Proto $scheme;
}
```

Adjust CORS and CSP connect-src to allow /newapp if your app enforces them.

### 5) Ops tips

- Disable autoDeploy in production, deploy explicitly.
- Keep secrets out of logs; mask client_secret in logging.
- Set Connector limits (uploads): maxPostSize, maxSwallowSize.
- For rotation: update file or env, then restart Tomcat (or implement hot-reload if required).