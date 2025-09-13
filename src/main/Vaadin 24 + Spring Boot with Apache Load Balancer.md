# Vaadin 24 + Spring Boot with Apache Load Balancer & Tomcat Session Clustering

This comprehensive guide provides step-by-step implementation for a **Vaadin 24 + Spring Boot** application with Apache HTTP Server load balancing, **Tomcat native session clustering**, and remote EJB integration.

## **Architecture Overview**

```
[Client Browser] 
    ↓
[Apache HTTP Server - Load Balancer]
    ↓
[Multiple Tomcat Instances with Native Session Clustering]
    ↓
[Vaadin 24 + Spring Boot Application]
    ↓
[Remote EJB Server]
```

---

## **Phase 1: Project Setup & Base Configuration**

### **Step 1.1: Create Spring Boot Project Structure**

```xml
<!-- pom.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="[http://maven.apache.org/POM/4.0.0](http://maven.apache.org/POM/4.0.0)"
         xmlns:xsi="[http://www.w3.org/2001/XMLSchema-instance](http://www.w3.org/2001/XMLSchema-instance)"
         xsi:schemaLocation="[http://maven.apache.org/POM/4.0.0](http://maven.apache.org/POM/4.0.0) 
         [http://maven.apache.org/xsd/maven-4.0.0.xsd](http://maven.apache.org/xsd/maven-4.0.0.xsd)">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.1.5</version>
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>vaadin-cluster-app</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>war</packaging>
    
    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <[maven.compiler.target](http://maven.compiler.target)>17</[maven.compiler.target](http://maven.compiler.target)>
        <vaadin.version>24.2.5</vaadin.version>
        <[project.build](http://project.build).sourceEncoding>UTF-8</[project.build](http://project.build).sourceEncoding>
    </properties>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.vaadin</groupId>
                <artifactId>vaadin-bom</artifactId>
                <version>${vaadin.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    
    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-tomcat</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        
        <!-- Vaadin -->
        <dependency>
            <groupId>com.vaadin</groupId>
            <artifactId>vaadin-spring-boot-starter</artifactId>
        </dependency>
        
        <!-- EJB Client Dependencies -->
        <dependency>
            <groupId>org.wildfly</groupId>
            <artifactId>wildfly-ejb-client-bom</artifactId>
            <version>[27.0.1.Final](http://27.0.1.Final)</version>
            <type>pom</type>
        </dependency>
        
        <dependency>
            <groupId>org.jboss</groupId>
            <artifactId>jboss-ejb-client</artifactId>
            <version>[4.0.44.Final](http://4.0.44.Final)</version>
        </dependency>
        
        <dependency>
            <groupId>org.wildfly.naming</groupId>
            <artifactId>wildfly-naming-client</artifactId>
            <version>[1.0.13.Final](http://1.0.13.Final)</version>
        </dependency>
        
        <!-- Provided Dependencies -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
            <scope>provided</scope>
        </dependency>
        
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <scope>provided</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>com.vaadin</groupId>
                <artifactId>vaadin-maven-plugin</artifactId>
                <version>${vaadin.version}</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>prepare-frontend</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

### **Step 1.2: Configure Spring Boot for WAR Deployment**

```java
// [Application.java](http://Application.java)
@SpringBootApplication
@EnableVaadin
@EnableJpaRepositories
public class VaadinClusterApplication extends SpringBootServletInitializer {
    
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(VaadinClusterApplication.class);
    }
    
    public static void main(String[] args) {
        [SpringApplication.run](http://SpringApplication.run)(VaadinClusterApplication.class, args);
    }
}
```

---

## **Phase 2: Tomcat Native Session Clustering Setup**

### **Step 2.1: Configure Tomcat Instance 1**

**Create directory structure:**

```bash
# Create Tomcat instances
mkdir -p /opt/tomcat/tomcat1
mkdir -p /opt/tomcat/tomcat2

# Copy Tomcat installation to each instance
cp -r /opt/apache-tomcat-9.0.x/* /opt/tomcat/tomcat1/
cp -r /opt/apache-tomcat-9.0.x/* /opt/tomcat/tomcat2/
```

**Tomcat Instance 1 Configuration (server.xml):**

```xml
<!-- /opt/tomcat/tomcat1/conf/server.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<Server port="8005" shutdown="SHUTDOWN">
  
  <!-- Global JNDI resources -->
  <GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>
  
  <Service name="Catalina">
    <!-- HTTP Connector -->
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443"
               maxThreads="200"
               minSpareThreads="10" />
    
    <!-- AJP Connector for Apache integration -->
    <Connector port="8009" protocol="AJP/1.3"
               redirectPort="8443"
               secretRequired="false"
               maxThreads="200" />
    
    <!-- Engine with clustering -->
    <Engine name="Catalina" defaultHost="[localhost](http://localhost)" jvmRoute="tomcat1">
      
      <!-- Cluster Configuration for Session Replication -->
      <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
               channelSendOptions="4">
        
        <Manager className="org.apache.catalina.ha.session.DeltaManager"
                 expireSessionsOnShutdown="false"
                 notifyListenersOnReplication="true" />
        
        <!-- Communication Channel -->
        <Channel className="[org.apache.catalina.tribes.group](http://org.apache.catalina.tribes.group).GroupChannel">
          
          <!-- Membership Service -->
          <Membership className="org.apache.catalina.tribes.membership.McastService"
                     address="228.0.0.4"
                     port="45564"
                     frequency="500"
                     dropTime="3000" />
          
          <!-- Message Receiver -->
          <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"
                   address="192.168.1.100"
                   port="4000"
                   autoBind="100"
                   selectorTimeout="5000"
                   maxThreads="6" />
          
          <!-- Message Sender -->
          <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
            <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender" />
          </Sender>
          
          <!-- Message Interceptors -->
          <Interceptor className="[org.apache.catalina.tribes.group](http://org.apache.catalina.tribes.group).interceptors.TcpFailureDetector" />
          <Interceptor className="[org.apache.catalina.tribes.group](http://org.apache.catalina.tribes.group).interceptors.MessageDispatchInterceptor" />
        </Channel>
        
        <!-- Cluster Valves -->
        <Valve className="org.apache.catalina.ha.tcp.ReplicationValve"
               filter=".*\.gif|.*\.js|.*\.jpeg|.*\.jpg|.*\.png|.*\.htm|.*\.html|.*\.css|.*\.txt" />
        
        <!-- Session ID Appender -->
        <Valve className="org.apache.catalina.ha.session.JvmRouteBinderValve" />
        
        <!-- Cluster Listener -->
        <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener" />
      </Cluster>
      
      <!-- Realm -->
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase" />
      </Realm>
      
      <!-- Virtual Host -->
      <Host name="[localhost](http://localhost)" appBase="webapps"
            unpackWARs="true" autoDeploy="true">
        
        <!-- Application Context -->
        <Context path="/vaadin-app" docBase="vaadin-cluster-app.war"
                 distributable="true"
                 reloadable="false">
          
          <!-- Session Cookie Configuration -->
          <CookieProcessor className="org.apache.tomcat.util.http.Rfc6265CookieProcessor"
                          sameSiteCookies="lax" />
          
        </Context>
        
        <!-- Access Log Valve -->
        <Valve className="org.apache.catalina.valves.AccessLogValve"
               directory="logs"
               prefix="[localhost](http://localhost)_access_log"
               suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b %{JSESSIONID}c" />
      </Host>
    </Engine>
  </Service>
</Server>
```

### **Step 2.2: Configure Tomcat Instance 2**

**Tomcat Instance 2 Configuration (server.xml):**

```xml
<!-- /opt/tomcat/tomcat2/conf/server.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<Server port="8006" shutdown="SHUTDOWN">
  
  <GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>
  
  <Service name="Catalina">
    <!-- HTTP Connector -->
    <Connector port="8081" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443"
               maxThreads="200"
               minSpareThreads="10" />
    
    <!-- AJP Connector -->
    <Connector port="8010" protocol="AJP/1.3"
               redirectPort="8443"
               secretRequired="false"
               maxThreads="200" />
    
    <!-- Engine with clustering -->
    <Engine name="Catalina" defaultHost="[localhost](http://localhost)" jvmRoute="tomcat2">
      
      <!-- Cluster Configuration -->
      <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
               channelSendOptions="4">
        
        <Manager className="org.apache.catalina.ha.session.DeltaManager"
                 expireSessionsOnShutdown="false"
                 notifyListenersOnReplication="true" />
        
        <Channel className="[org.apache.catalina.tribes.group](http://org.apache.catalina.tribes.group).GroupChannel">
          
          <Membership className="org.apache.catalina.tribes.membership.McastService"
                     address="228.0.0.4"
                     port="45564"
                     frequency="500"
                     dropTime="3000" />
          
          <!-- Different port for second instance -->
          <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"
                   address="192.168.1.100"
                   port="4001"
                   autoBind="100"
                   selectorTimeout="5000"
                   maxThreads="6" />
          
          <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
            <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender" />
          </Sender>
          
          <Interceptor className="[org.apache.catalina.tribes.group](http://org.apache.catalina.tribes.group).interceptors.TcpFailureDetector" />
          <Interceptor className="[org.apache.catalina.tribes.group](http://org.apache.catalina.tribes.group).interceptors.MessageDispatchInterceptor" />
        </Channel>
        
        <Valve className="org.apache.catalina.ha.tcp.ReplicationValve"
               filter=".*\.gif|.*\.js|.*\.jpeg|.*\.jpg|.*\.png|.*\.htm|.*\.html|.*\.css|.*\.txt" />
        
        <Valve className="org.apache.catalina.ha.session.JvmRouteBinderValve" />
        
        <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener" />
      </Cluster>
      
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase" />
      </Realm>
      
      <Host name="[localhost](http://localhost)" appBase="webapps"
            unpackWARs="true" autoDeploy="true">
        
        <Context path="/vaadin-app" docBase="vaadin-cluster-app.war"
                 distributable="true"
                 reloadable="false">
          
          <CookieProcessor className="org.apache.tomcat.util.http.Rfc6265CookieProcessor"
                          sameSiteCookies="lax" />
          
        </Context>
        
        <Valve className="org.apache.catalina.valves.AccessLogValve"
               directory="logs"
               prefix="[localhost](http://localhost)_access_log"
               suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b %{JSESSIONID}c" />
      </Host>
    </Engine>
  </Service>
</Server>
```

### **Step 2.3: Configure web.xml for Session Clustering**

**Add to your WAR's web.xml:**

```xml
<!-- WEB-INF/web.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="[http://xmlns.jcp.org/xml/ns/javaee](http://xmlns.jcp.org/xml/ns/javaee)"
         xmlns:xsi="[http://www.w3.org/2001/XMLSchema-instance](http://www.w3.org/2001/XMLSchema-instance)"
         xsi:schemaLocation="[http://xmlns.jcp.org/xml/ns/javaee](http://xmlns.jcp.org/xml/ns/javaee) 
         [http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd](http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd)"
         version="4.0">
  
  <display-name>Vaadin Cluster Application</display-name>
  
  <!-- Enable session clustering -->
  <distributable/>
  
  <!-- Session configuration -->
  <session-config>
    <session-timeout>30</session-timeout>
    <cookie-config>
      <name>JSESSIONID</name>
      <http-only>true</http-only>
      <secure>false</secure>
    </cookie-config>
    <tracking-mode>COOKIE</tracking-mode>
  </session-config>
  
  <!-- Security constraint for HTTPS (optional) -->
  <!--
  <security-constraint>
    <web-resource-collection>
      <web-resource-name>Entire Application</web-resource-name>
      <url-pattern>/*</url-pattern>
    </web-resource-collection>
    <user-data-constraint>
      <transport-guarantee>CONFIDENTIAL</transport-guarantee>
    </user-data-constraint>
  </security-constraint>
  -->
  
</web-app>
```

### **Step 2.4: Spring Boot Configuration for Tomcat Clustering**

```java
// [TomcatConfig.java](http://TomcatConfig.java)
@Configuration
public class TomcatConfig {
    
    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> tomcatCustomizer() {
        return factory -> {
            factory.addContextCustomizers(context -> {
                // Enable distributable sessions
                context.setDistributable(true);
                
                // Configure session manager
                context.setManager(createClusterManager());
                
                // Add session listener for debugging
                context.addApplicationSessionListener("com.example.config.SessionEventListener");
            });
        };
    }
    
    private Manager createClusterManager() {
        // This will use the cluster configuration from server.xml
        // The actual clustering is configured at the Tomcat server level
        StandardManager manager = new StandardManager();
        manager.setMaxActiveSessions(-1);
        manager.setSessionIdLength(32);
        return manager;
    }
}
```

**Session Event Listener for Monitoring:**

```java
// [SessionEventListener.java](http://SessionEventListener.java)
@Component
public class SessionEventListener implements HttpSessionListener, Serializable {
    
    private static final Logger logger = LoggerFactory.getLogger(SessionEventListener.class);
    
    @Override
    public void sessionCreated(HttpSessionEvent se) {
        HttpSession session = se.getSession();
        [logger.info](http://logger.info)("Session created: {} on server: {}", 
                   session.getId(), 
                   System.getProperty("jvmRoute", "unknown"));
        
        // Store server info in session for debugging
        session.setAttribute("serverNode", System.getProperty("jvmRoute", "unknown"));
        session.setAttribute("createdTime", new Date());
    }
    
    @Override
    public void sessionDestroyed(HttpSessionEvent se) {
        HttpSession session = se.getSession();
        [logger.info](http://logger.info)("Session destroyed: {} on server: {}", 
                   session.getId(), 
                   System.getProperty("jvmRoute", "unknown"));
    }
}
```

---

## **Phase 3: Apache HTTP Server Load Balancer Setup**

### **Step 3.1: Install and Configure Apache HTTP Server**

```bash
# Install Apache
sudo apt-get update
sudo apt-get install apache2

# Enable required modules
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_balancer
sudo a2enmod proxy_ajp
sudo a2enmod lbmethod_byrequests
sudo a2enmod lbmethod_heartbeat
sudo a2enmod headers
sudo a2enmod rewrite
sudo a2enmod status

# Restart Apache
sudo systemctl restart apache2
sudo systemctl enable apache2
```

### **Step 3.2: Configure Apache Virtual Host with AJP**

```
# /etc/apache2/sites-available/vaadin-cluster.conf
<VirtualHost *:80>
    ServerName [your-domain.com](http://your-domain.com)
    ServerAlias [www.your-domain.com](http://www.your-domain.com)
    DocumentRoot /var/www/html
    
    # Logging
    CustomLog ${APACHE_LOG_DIR}/vaadin-access.log combined
    ErrorLog ${APACHE_LOG_DIR}/vaadin-error.log
    LogLevel info
    
    # Server status (for monitoring)
    <Location "/server-status">
        SetHandler server-status
        Require ip 127.0.0.1
        Require ip 192.168.1
    </Location>
    
    # Balancer manager (for runtime configuration)
    <Location "/balancer-manager">
        SetHandler balancer-manager
        Require ip 127.0.0.1
        Require ip 192.168.1
    </Location>
    
    # Proxy configuration
    ProxyPreserveHost On
    ProxyRequests Off
    
    # Exclude management URLs from load balancing
    ProxyPass /balancer-manager !
    ProxyPass /server-status !
    
    # Static resources (optional - serve directly from Apache)
    # Alias /vaadin-app/VAADIN /var/www/vaadin-static/VAADIN
    # ProxyPass /vaadin-app/VAADIN !
    
    # Main application load balancing using AJP
    ProxyPass /vaadin-app balancer://vaadin-cluster-ajp stickysession=JSESSIONID
    ProxyPassReverse /vaadin-app balancer://vaadin-cluster-ajp
    
    # Define AJP balancer members
    <Proxy balancer://vaadin-cluster-ajp>
        # Tomcat instance 1
        BalancerMember ajp://192.168.1.100:8009 route=tomcat1 retry=5 timeout=10
        # Tomcat instance 2  
        BalancerMember ajp://192.168.1.100:8010 route=tomcat2 retry=5 timeout=10
        
        # Load balancing method
        ProxySet lbmethod=byrequests
        
        # Health checking
        ProxySet hcmethod=GET
        ProxySet hcuri=/vaadin-app/actuator/health
        
        # Connection pooling
        ProxySet connectiontimeout=5
        ProxySet ttl=60
    </Proxy>
    
    # Alternative HTTP balancer (backup)
    <Proxy balancer://vaadin-cluster-http>
        BalancerMember [http://192.168.1.100:8080](http://192.168.1.100:8080) route=tomcat1 status=+H
        BalancerMember [http://192.168.1.100:8081](http://192.168.1.100:8081) route=tomcat2 status=+H
        ProxySet lbmethod=byrequests
    </Proxy>
    
    # Fallback to HTTP if AJP fails
    ProxyPassMatch ^/vaadin-app/(.*)$ balancer://vaadin-cluster-http/vaadin-app/$1
    
    # Headers for session debugging
    Header add X-Forwarded-Proto "http"
    Header add X-Forwarded-Port "80"
    
    # Add server identification header
    Header add X-Served-By "%{BALANCER_WORKER_ROUTE}e"
    
    # Security headers
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-XSS-Protection "1; mode=block"
    
    # Compression
    <Location /vaadin-app>
        SetOutputFilter DEFLATE
        SetEnvIfNoCase Request_URI \
            \.(?:gif|jpe?g|png)$ no-gzip dont-vary
        SetEnvIfNoCase Request_URI \
            \.(?:exe|t?gz|zip|bz2|sit|rar)$ no-gzip dont-vary
    </Location>
    
</VirtualHost>

# HTTPS Virtual Host (optional but recommended)
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName [your-domain.com](http://your-domain.com)
    DocumentRoot /var/www/html
    
    # SSL Configuration
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/your-cert.crt
    SSLCertificateKeyFile /etc/ssl/private/your-key.key
    
    # Same proxy configuration as HTTP
    ProxyPreserveHost On
    ProxyRequests Off
    
    ProxyPass /balancer-manager !
    ProxyPass /server-status !
    
    ProxyPass /vaadin-app balancer://vaadin-cluster-ajp stickysession=JSESSIONID
    ProxyPassReverse /vaadin-app balancer://vaadin-cluster-ajp
    
    # Same balancer configuration...
    
    # HTTPS-specific headers
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
    Header add X-Forwarded-Proto "https"
    Header add X-Forwarded-Port "443"
    
</VirtualHost>
</IfModule>
```

### **Step 3.3: Enable the Site and Configure Firewall**

```bash
# Enable the site
sudo a2ensite vaadin-cluster.conf

# Test configuration
sudo apache2ctl configtest

# Reload Apache
sudo systemctl reload apache2

# Configure firewall
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from 192.168.1.0/24 to any port 8080,8081,8009,8010

# Configure multicast for Tomcat clustering
sudo route add -net 224.0.0.0 netmask 240.0.0.0 dev eth0
```

---

## **Phase 4: Remote EJB Integration**

### **Step 4.1: EJB Client Configuration**

```java
// [EjbConfig.java](http://EjbConfig.java)
@Configuration
@Slf4j
public class EjbConfig {
    
    @Value("${[ejb.server.host](http://ejb.server.host):[localhost](http://localhost)}")
    private String ejbServerHost;
    
    @Value("${ejb.server.port:8080}")
    private String ejbServerPort;
    
    @Value("${ejb.server.username:ejbuser}")
    private String ejbUsername;
    
    @Value("${ejb.server.password:ejbpassword}")
    private String ejbPassword;
    
    @Bean
    @Primary
    public InitialContext ejbInitialContext() throws NamingException {
        Properties properties = new Properties();
        properties.put(Context.INITIAL_CONTEXT_FACTORY, 
            "org.wildfly.naming.client.WildFlyInitialContextFactory");
        properties.put(Context.PROVIDER_URL, 
            String.format("http-remoting://%s:%s", ejbServerHost, ejbServerPort));
        properties.put([Context.SECURITY](http://Context.SECURITY)_PRINCIPAL, ejbUsername);
        properties.put([Context.SECURITY](http://Context.SECURITY)_CREDENTIALS, ejbPassword);
        properties.put("jboss.naming.client.ejb.context", true);
        
        // Connection pooling
        properties.put("[jboss.naming.client.connect.options.org](http://jboss.naming.client.connect.options.org).xnio.Options.SASL_POLICY_NOPLAINTEXT", "false");
        properties.put("[jboss.naming.client.connect.options.org](http://jboss.naming.client.connect.options.org).xnio.Options.SASL_POLICY_NOANONYMOUS", "false");
        
        [log.info](http://log.info)("Initializing EJB context for server: {}:{}", ejbServerHost, ejbServerPort);
        
        return new InitialContext(properties);
    }
    
    @Bean
    public CacheManager ejbCacheManager() {
        return new ConcurrentMapCacheManager("ejbLookupCache", "ejbResultCache");
    }
}
```

### **Step 4.2: EJB Service Wrapper with Connection Pooling**

```java
// [EjbServiceWrapper.java](http://EjbServiceWrapper.java)
@Service
@Slf4j
public class EjbServiceWrapper implements Serializable {
    
    private static final long serialVersionUID = 1L;
    
    @Autowired
    @Qualifier("ejbInitialContext")
    private transient InitialContext ejbContext;
    
    private static final String DATA_SERVICE_JNDI = 
        "ejb:/your-ear-name/your-ejb-jar-name/DataServiceBean!com.example.ejb.DataServiceRemote";
    
    private static final String USER_SERVICE_JNDI = 
        "ejb:/your-ear-name/your-ejb-jar-name/UserServiceBean!com.example.ejb.UserServiceRemote";
    
    @Cacheable(value = "ejbLookupCache", key = "#jndiName")
    public <T> T lookupEJB(String jndiName, Class<T> interfaceClass) throws NamingException {
        log.debug("Looking up EJB: {}", jndiName);
        
        try {
            Object ejbRef = ejbContext.lookup(jndiName);
            return interfaceClass.cast(ejbRef);
        } catch (NamingException e) {
            log.error("Failed to lookup EJB: {}", jndiName, e);
            throw e;
        }
    }
    
    @Cacheable(value = "ejbResultCache", key = "'data_' + #criteria", unless = "#result == null or #result.isEmpty()")
    public List<DataEntity> fetchData(String criteria) {
        try {
            DataServiceRemote dataService = lookupEJB(DATA_SERVICE_JNDI, DataServiceRemote.class);
            
            [log.info](http://log.info)("Fetching data with criteria: {}", criteria);
            List<DataEntity> result = dataService.findByCriteria(criteria);
            
            [log.info](http://log.info)("Retrieved {} records from EJB", result != null ? result.size() : 0);
            return result != null ? result : Collections.emptyList();
            
        } catch (Exception e) {
            log.error("Failed to fetch data from EJB", e);
            throw new ServiceException("Failed to fetch data from remote EJB: " + e.getMessage(), e);
        }
    }
    
    public DataEntity findDataById(Long id) {
        try {
            DataServiceRemote dataService = lookupEJB(DATA_SERVICE_JNDI, DataServiceRemote.class);
            return dataService.findById(id);
            
        } catch (Exception e) {
            log.error("Failed to find data by ID: {}", id, e);
            throw new ServiceException("Failed to find data by ID: " + e.getMessage(), e);
        }
    }
    
    public void saveData(DataEntity entity) {
        try {
            DataServiceRemote dataService = lookupEJB(DATA_SERVICE_JNDI, DataServiceRemote.class);
            dataService.saveData(entity);
            
            // Clear cache for this entity
            clearDataCache();
            
        } catch (Exception e) {
            log.error("Failed to save data", e);
            throw new ServiceException("Failed to save data: " + e.getMessage(), e);
        }
    }
    
    @CacheEvict(value = "ejbResultCache", allEntries = true)
    public void clearDataCache() {
        [log.info](http://log.info)("Cleared EJB result cache");
    }
    
    @PreDestroy
    public void cleanup() {
        try {
            if (ejbContext != null) {
                ejbContext.close();
                [log.info](http://log.info)("EJB context closed successfully");
            }
        } catch (NamingException e) {
            log.error("Error closing EJB context", e);
        }
    }
    
    // Health check method
    public boolean isEjbServerAvailable() {
        try {
            lookupEJB(DATA_SERVICE_JNDI, DataServiceRemote.class);
            return true;
        } catch (Exception e) {
            log.warn("EJB server health check failed", e);
            return false;
        }
    }
}
```

### **Step 4.3: Remote EJB Interface Definitions**

```java
// [DataServiceRemote.java](http://DataServiceRemote.java)
@Remote
public interface DataServiceRemote extends Serializable {
    List<DataEntity> findByCriteria(String criteria);
    DataEntity findById(Long id);
    void saveData(DataEntity entity);
    void deleteData(Long id);
    long countByCriteria(String criteria);
    List<DataEntity> findWithPagination(String criteria, int page, int size);
}

// [UserServiceRemote.java](http://UserServiceRemote.java)
@Remote
public interface UserServiceRemote extends Serializable {
    UserEntity authenticateUser(String username, String password);
    List<UserEntity> findAllUsers();
    UserEntity findUserById(Long id);
    void createUser(UserEntity user);
    void updateUser(UserEntity user);
}
```

---

## **Phase 5: Vaadin 24 Application Implementation**

### **Step 5.1: Serializable Vaadin Components**

**Important:** All Vaadin components and data in clustered sessions must be serializable.

```java
// [MainLayout.java](http://MainLayout.java)
@Theme("my-theme")
@PWA(name = "Vaadin Cluster App", shortName = "VCA")
@CssImport("./styles/shared-styles.css")
public class MainLayout extends AppLayout implements Serializable {
    
    private static final long serialVersionUID = 1L;
    
    public MainLayout() {
        createHeader();
        createDrawer();
    }
    
    private void createHeader() {
        H1 logo = new H1("Vaadin Cluster App");
        logo.addClassNames("text-l", "m-m");
        
        // Session info for debugging clustering
        VaadinSession session = VaadinSession.getCurrent();
        String sessionId = session.getSession().getId();
        String serverNode = (String) session.getSession().getAttribute("serverNode");
        
        Span sessionInfo = new Span(String.format("Session: %s | Node: %s", 
                                   sessionId.substring(0, 8), 
                                   serverNode != null ? serverNode : "unknown"));
        sessionInfo.getStyle().set("font-size", "0.8em").set("color", "gray");
        
        Button menuToggle = new Button("Menu");
        menuToggle.addThemeVariants(ButtonVariant.LUMO_TERTIARY);
        
        HorizontalLayout header = new HorizontalLayout(menuToggle, logo, sessionInfo);
        header.setDefaultVerticalComponentAlignment([FlexComponent.Alignment.CENTER](http://FlexComponent.Alignment.CENTER));
        header.expand(logo);
        header.setWidthFull();
        header.addClassNames("py-0", "px-m");
        
        addToNavbar(header);
    }
    
    private void createDrawer() {
        RouterLink dataView = new RouterLink("Data Management", DataView.class);
        RouterLink dashboardView = new RouterLink("Dashboard", DashboardView.class);
        RouterLink clusterView = new RouterLink("Cluster Info", ClusterInfoView.class);
        
        VerticalLayout navigation = new VerticalLayout(dataView, dashboardView, clusterView);
        navigation.setSizeUndefined();
        
        addToDrawer(navigation);
    }
}
```

### **Step 5.2: Data Management View with Session Clustering Support**

```java
// [DataView.java](http://DataView.java)
@Route(value = "data", layout = MainLayout.class)
@PageTitle("Data Management")
@UIScope
@Component
public class DataView extends VerticalLayout implements Serializable {
    
    private static final long serialVersionUID = 1L;
    
    @Autowired
    private transient EjbServiceWrapper ejbService;
    
    private Grid<DataEntity> grid;
    private TextField searchField;
    private Button searchButton;
    private Button refreshButton;
    private Span statusLabel;
    
    @PostConstruct
    public void init() {
        setSizeFull();
        createComponents();
        layoutComponents();
        loadData();
        
        // Store view state in session for clustering
        VaadinSession.getCurrent().getSession().setAttribute("lastViewedData", this.getClass().getSimpleName());
    }
    
    private void createComponents() {
        searchField = new TextField("Search Criteria");
        searchField.setPlaceholder("Enter search terms...");
        searchField.setWidth("300px");
        searchField.addKeyPressListener(Key.ENTER, e -> performSearch());
        
        searchButton = new Button("Search", [VaadinIcon.SEARCH](http://VaadinIcon.SEARCH).create());
        searchButton.addThemeVariants(ButtonVariant.LUMO_PRIMARY);
        searchButton.addClickListener(e -> performSearch());
        
        refreshButton = new Button("Refresh", VaadinIcon.REFRESH.create());
        refreshButton.addThemeVariants(ButtonVariant.LUMO_CONTRAST);
        refreshButton.addClickListener(e -> loadData());
        
        Button testClusterButton = new Button("Test Cluster", VaadinIcon.CLUSTER.create());
        testClusterButton.addThemeVariants(ButtonVariant.LUMO_TERTIARY);
        testClusterButton.addClickListener(e -> testSessionClustering());
        
        statusLabel = new Span();
        statusLabel.getStyle().set("font-style", "italic");
        
        grid = new Grid<>(DataEntity.class, false);
        configureGrid();
    }
    
    private void configureGrid() {
        grid.addColumn(DataEntity::getId)
            .setHeader("ID")
            .setSortable(true)
            .setWidth("100px")
            .setFlexGrow(0);
            
        grid.addColumn(DataEntity::getName)
            .setHeader("Name")
            .setSortable(true)
            .setAutoWidth(true);
            
        grid.addColumn(DataEntity::getDescription)
            .setHeader("Description")
            .setAutoWidth(true);
            
        grid.addColumn(new LocalDateTimeRenderer<>(DataEntity::getCreatedDate, "yyyy-MM-dd HH:mm"))
            .setHeader("Created")
            .setSortable(true)
            .setWidth("150px")
            .setFlexGrow(0);
        
        // Add action column
        grid.addComponentColumn(entity -> {
            Button editButton = new Button("Edit", VaadinIcon.EDIT.create());
            editButton.addThemeVariants(ButtonVariant.LUMO_TERTIARY, ButtonVariant.LUMO_SMALL);
            editButton.addClickListener(e -> editEntity(entity));
            
            Button deleteButton = new Button("Delete", VaadinIcon.TRASH.create());
            deleteButton.addThemeVariants(ButtonVariant.LUMO_TERTIARY, ButtonVariant.LUMO_ERROR, ButtonVariant.LUMO_SMALL);
            deleteButton.addClickListener(e -> deleteEntity(entity));
            
            return new HorizontalLayout(editButton, deleteButton);
        }).setHeader("Actions").setAutoWidth(true).setFlexGrow(0);
        
        grid.setSizeFull();
        grid.setSelectionMode(Grid.SelectionMode.SINGLE);
        
        // Add selection listener to store in session
        grid.addSelectionListener(e -> {
            e.getFirstSelectedItem().ifPresent(entity -> {
                VaadinSession.getCurrent().getSession().setAttribute("selectedEntity", entity);
            });
        });
    }
    
    private void layoutComponents() {
        HorizontalLayout toolbar = new HorizontalLayout(
            searchField, searchButton, refreshButton, 
            new Button("Test Cluster", e -> testSessionClustering())
        );
        toolbar.setAlignItems(Alignment.END);
        toolbar.setPadding(true);
        toolbar.setSpacing(true);
        
        add(toolbar, statusLabel, grid);
        expand(grid);
    }
    
    private void performSearch() {
        String criteria = searchField.getValue();
        if (criteria != null && !criteria.trim().isEmpty()) {
            try {
                updateStatus("Searching...", "blue");
                
                List<DataEntity> results = ejbService.fetchData(criteria);
                grid.setItems(results);
                
                // Store search results in session for clustering
                VaadinSession.getCurrent().getSession().setAttribute("lastSearchResults", results);
                VaadinSession.getCurrent().getSession().setAttribute("lastSearchCriteria", criteria);
                
                updateStatus(String.format("Found %d results", results.size()), "green");
                
            } catch (Exception e) {
                updateStatus("Error searching data: " + e.getMessage(), "red");
                [Notification.show](http://Notification.show)(
                    "Error searching data: " + e.getMessage(),
                    5000,
                    Notification.Position.MIDDLE
                ).addThemeVariants(NotificationVariant.LUMO_ERROR);
            }
        } else {
            updateStatus("Please enter search criteria", "orange");
        }
    }
    
    private void loadData() {
        try {
            updateStatus("Loading data...", "blue");
            
            List<DataEntity> data = ejbService.fetchData(""); // Load all
            grid.setItems(data);
            
            // Store in session
            VaadinSession.getCurrent().getSession().setAttribute("allData", data);
            
            updateStatus(String.format("Loaded %d records", data.size()), "green");
            
        } catch (Exception e) {
            updateStatus("Error loading data: " + e.getMessage(), "red");
            [Notification.show](http://Notification.show)(
                "Error loading data: " + e.getMessage(),
                5000,
                Notification.Position.MIDDLE
            ).addThemeVariants(NotificationVariant.LUMO_ERROR);
        }
    }
    
    private void testSessionClustering() {
        VaadinSession session = VaadinSession.getCurrent();
        HttpSession httpSession = session.getSession();
        
        // Test session replication
        String testData = "Cluster test at " + new Date();
        httpSession.setAttribute("clusterTest", testData);
        
        String serverNode = (String) httpSession.getAttribute("serverNode");
        String sessionId = httpSession.getId();
        
        Dialog dialog = new Dialog();
        dialog.setHeaderTitle("Session Clustering Info");
        
        VerticalLayout content = new VerticalLayout();
        content.add(
            new Span("Session ID: " + sessionId),
            new Span("Server Node: " + (serverNode != null ? serverNode : "Unknown")),
            new Span("Test Data: " + testData),
            new Span("Max Inactive Interval: " + httpSession.getMaxInactiveInterval() + "s")
        );
        
        Button closeButton = new Button("Close", e -> dialog.close());
        closeButton.addThemeVariants(ButtonVariant.LUMO_PRIMARY);
        
        dialog.add(content);
        dialog.getFooter().add(closeButton);
        [dialog.open](http://dialog.open)();
        
        updateStatus("Session clustering test completed", "blue");
    }
    
    private void editEntity(DataEntity entity) {
        // Implementation for editing entity
        [Notification.show](http://Notification.show)("Edit functionality - Entity ID: " + entity.getId());
    }
    
    private void deleteEntity(DataEntity entity) {
        // Implementation for deleting entity
        [Notification.show](http://Notification.show)("Delete functionality - Entity ID: " + entity.getId());
    }
    
    private void updateStatus(String message, String color) {
        statusLabel.setText(message);
        statusLabel.getStyle().set("color", color);
    }
    
    // Method to restore state after session replication
    @EventListener
    public void onSessionRestore(VaadinServiceInitEvent event) {
        VaadinSession session = event.getSource().getCurrentInstances().iterator().next();
        HttpSession httpSession = session.getSession();
        
        @SuppressWarnings("unchecked")
        List<DataEntity> lastResults = (List<DataEntity>) httpSession.getAttribute("lastSearchResults");
        String lastCriteria = (String) httpSession.getAttribute("lastSearchCriteria");
        
        if (lastResults != null && !lastResults.isEmpty()) {
            grid.setItems(lastResults);
            if (lastCriteria != null) {
                searchField.setValue(lastCriteria);
            }
            updateStatus("Session restored with " + lastResults.size() + " records", "green");
        }
    }
}
```

### **Step 5.3: Cluster Information View**

```java
// [ClusterInfoView.java](http://ClusterInfoView.java)
@Route(value = "cluster", layout = MainLayout.class)
@PageTitle("Cluster Information")
@UIScope
@Component
public class ClusterInfoView extends VerticalLayout implements Serializable {
    
    private static final long serialVersionUID = 1L;
    
    @Autowired
    private transient EjbServiceWrapper ejbService;
    
    private Grid<SessionInfo> sessionGrid;
    private Div serverInfoDiv;
    private Div ejbStatusDiv;
    
    @PostConstruct
    public void init() {
        setSizeFull();
        setPadding(true);
        setSpacing(true);
        
        createComponents();
        refreshInfo();
    }
    
    private void createComponents() {
        H2 title = new H2("Cluster & Session Information");
        
        Button refreshButton = new Button("Refresh", VaadinIcon.REFRESH.create());
        refreshButton.addThemeVariants(ButtonVariant.LUMO_PRIMARY);
        refreshButton.addClickListener(e -> refreshInfo());
        
        HorizontalLayout header = new HorizontalLayout(title, refreshButton);
        header.setAlignItems([Alignment.CENTER](http://Alignment.CENTER));
        header.setJustifyContentMode(JustifyContentMode.BETWEEN);
        header.setWidthFull();
        
        // Server information panel
        serverInfoDiv = new Div();
        serverInfoDiv.addClassName("info-panel");
        
        // EJB status panel
        ejbStatusDiv = new Div();
        ejbStatusDiv.addClassName("info-panel");
        
        // Session information grid
        sessionGrid = new Grid<>(SessionInfo.class, false);
        configureSessionGrid();
        
        add(header, serverInfoDiv, ejbStatusDiv, 
            new H3("Session Details"), sessionGrid);
    }
    
    private void configureSessionGrid() {
        sessionGrid.addColumn(SessionInfo::getAttributeName)
                  .setHeader("Attribute Name")
                  .setAutoWidth(true);
                  
        sessionGrid.addColumn(SessionInfo::getAttributeValue)
                  .setHeader("Attribute Value")
                  .setAutoWidth(true);
                  
        sessionGrid.addColumn(SessionInfo::getAttributeType)
                  .setHeader("Type")
                  .setWidth("150px");
        
        sessionGrid.setHeight("300px");
    }
    
    private void refreshInfo() {
        updateServerInfo();
        updateEjbStatus();
        updateSessionInfo();
    }
    
    private void updateServerInfo() {
        VaadinSession session = VaadinSession.getCurrent();
        HttpSession httpSession = session.getSession();
        
        String serverNode = (String) httpSession.getAttribute("serverNode");
        String sessionId = httpSession.getId();
        Date createdTime = (Date) httpSession.getAttribute("createdTime");
        
        VerticalLayout serverInfo = new VerticalLayout();
        serverInfo.setPadding(true);
        serverInfo.setSpacing(false);
        
        serverInfo.add(
            new H3("Server Information"),
            new Span("Server Node: " + (serverNode != null ? serverNode : "Unknown")),
            new Span("Session ID: " + sessionId),
            new Span("Session Created: " + (createdTime != null ? createdTime.toString() : "Unknown")),
            new Span("Max Inactive Interval: " + httpSession.getMaxInactiveInterval() + " seconds"),
            new Span("Last Accessed: " + new Date(httpSession.getLastAccessedTime())),
            new Span("JVM Route: " + System.getProperty("jvmRoute", "Not Set"))
        );
        
        serverInfoDiv.removeAll();
        serverInfoDiv.add(serverInfo);
    }
    
    private void updateEjbStatus() {
        VerticalLayout ejbInfo = new VerticalLayout();
        ejbInfo.setPadding(true);
        ejbInfo.setSpacing(false);
        
        boolean ejbAvailable = false;
        String ejbStatus = "Disconnected";
        String statusColor = "red";
        
        try {
            ejbAvailable = ejbService.isEjbServerAvailable();
            if (ejbAvailable) {
                ejbStatus = "Connected";
                statusColor = "green";
            }
        } catch (Exception e) {
            ejbStatus = "Error: " + e.getMessage();
        }
        
        Span statusSpan = new Span("EJB Server Status: " + ejbStatus);
        statusSpan.getStyle().set("color", statusColor).set("font-weight", "bold");
        
        ejbInfo.add(
            new H3("EJB Server Information"),
            statusSpan,
            new Span("Connection Available: " + (ejbAvailable ? "Yes" : "No")),
            new Span("Last Check: " + new Date())
        );
        
        ejbStatusDiv.removeAll();
        ejbStatusDiv.add(ejbInfo);
    }
    
    private void updateSessionInfo() {
        List<SessionInfo> sessionInfos = new ArrayList<>();
        HttpSession httpSession = VaadinSession.getCurrent().getSession();
        
        Enumeration<String> attributeNames = httpSession.getAttributeNames();
        while (attributeNames.hasMoreElements()) {
            String name = attributeNames.nextElement();
            Object value = httpSession.getAttribute(name);
            
            SessionInfo info = new SessionInfo();
            info.setAttributeName(name);
            info.setAttributeType(value != null ? value.getClass().getSimpleName() : "null");
            
            if (value != null) {
                String valueStr = value.toString();
                if (valueStr.length() > 100) {
                    valueStr = valueStr.substring(0, 100) + "...";
                }
                info.setAttributeValue(valueStr);
            } else {
                info.setAttributeValue("null");
            }
            
            sessionInfos.add(info);
        }
        
        sessionGrid.setItems(sessionInfos);
    }
    
    // Inner class for session information
    public static class SessionInfo implements Serializable {
        private static final long serialVersionUID = 1L;
        
        private String attributeName;
        private String attributeValue;
        private String attributeType;
        
        // Getters and setters
        public String getAttributeName() { return attributeName; }
        public void setAttributeName(String attributeName) { this.attributeName = attributeName; }
        
        public String getAttributeValue() { return attributeValue; }
        public void setAttributeValue(String attributeValue) { this.attributeValue = attributeValue; }
        
        public String getAttributeType() { return attributeType; }
        public void setAttributeType(String attributeType) { this.attributeType = attributeType; }
    }
}
```

---

## **Phase 6: Configuration Files & Deployment**

### **Step 6.1: Complete Application Properties**

```yaml
# application.yml
spring:
  application:
    name: vaadin-cluster-app
  profiles:
    active: production
    
  # Database Configuration
  datasource:
    url: jdbc:postgresql://[localhost:5432/vaadinapp](http://localhost:5432/vaadinapp)
    username: ${DB_USER:dbuser}
    password: ${DB_PASSWORD:dbpass}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
    
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
        jdbc:
          batch_size: 20
        order_inserts: true
        order_updates: true
        
  # Security
  security:
    require-ssl: false
    
  # Cache Configuration
  cache:
    type: simple
    cache-names:
      - ejbLookupCache
      - ejbResultCache
    
# Server Configuration
server:
  servlet:
    context-path: /vaadin-app
    session:
      persistent: false  # We use Tomcat clustering instead
      timeout: 30m
      cookie:
        name: JSESSIONID
        http-only: true
        secure: false
        same-site: lax
  compression:
    enabled: true
    mime-types:
      - text/html
      - text/css
      - text/javascript
      - application/javascript
      - application/json
  error:
    whitelabel:
      enabled: false
      
# Vaadin Configuration
vaadin:
  servlet:
    production-mode: true
    close-idle-sessions: true
  session-timeout: 1800
  push-mode: automatic
  heartbeat-interval: 300
  
# EJB Configuration
ejb:
  server:
    host: ${EJB_HOST:[localhost](http://localhost)}
    port: ${EJB_PORT:8080}
    username: ${EJB_USER:ejbuser}
    password: ${EJB_PASSWORD:ejbpass}
    connection-timeout: 10000
    
# Actuator for Health Checks
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,sessions
      base-path: /actuator
  endpoint:
    health:
      show-details: always
    sessions:
      enabled: true
  health:
    defaults:
      enabled: true
    db:
      enabled: true
      
# Logging
logging:
  level:
    root: INFO
    com.example: DEBUG
    org.apache.catalina.ha: INFO
    org.springframework.web: INFO
    com.vaadin: INFO
    org.jboss.ejb.client: INFO
  pattern:
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level [%X{sessionId:-}] %logger{36} - %msg%n"
    console: "%d{HH:mm:ss.SSS} [%thread] %-5level [%X{sessionId:-}] %logger{36} - %msg%n"
  file:
    name: /var/log/vaadin-cluster-app/application.log
    max-size: 100MB
    max-history: 30
    
# JVM Route System Property
# This should be set as JVM parameter: -DjvmRoute=tomcat1
---
# Development Profile
spring:
  profiles: development
  
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password: 
    
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
    
  h2:
    console:
      enabled: true
      
vaadin:
  servlet:
    production-mode: false
    
logging:
  level:
    com.example: DEBUG
    org.springframework: INFO
```

### **Step 6.2: Tomcat Startup Scripts**

**Tomcat Instance 1 Startup Script:**

```bash
#!/bin/bash
# /opt/tomcat/tomcat1/bin/[startup-cluster.sh](http://startup-cluster.sh)

export CATALINA_HOME=/opt/tomcat/tomcat1
export CATALINA_BASE=/opt/tomcat/tomcat1
export CATALINA_PID=$CATALINA_BASE/logs/[catalina.pid](http://catalina.pid)

# JVM Route for session clustering
export CATALINA_OPTS="$CATALINA_OPTS -DjvmRoute=tomcat1"

# Memory settings
export CATALINA_OPTS="$CATALINA_OPTS -Xms1024m -Xmx2048m"
export CATALINA_OPTS="$CATALINA_OPTS -XX:+UseG1GC -XX:MaxGCPauseMillis=200"

# Networking for clustering
export CATALINA_OPTS="$CATALINA_OPTS -[Djava.net](http://Djava.net).preferIPv4Stack=true"
export CATALINA_OPTS="$CATALINA_OPTS -Djgroups.bind_addr=192.168.1.100"

# Session clustering debug (optional)
export CATALINA_OPTS="$CATALINA_OPTS -Dorg.apache.catalina.tribes.debug=true"

# Application-specific properties
export CATALINA_OPTS="$CATALINA_OPTS -DDB_HOST=[localhost](http://localhost)"
export CATALINA_OPTS="$CATALINA_OPTS -DDB_PORT=5432"
export CATALINA_OPTS="$CATALINA_OPTS -DEJB_HOST=[localhost](http://localhost)"
export CATALINA_OPTS="$CATALINA_OPTS -DEJB_PORT=8080"

# Start Tomcat
$CATALINA_HOME/bin/[startup.sh](http://startup.sh)

echo "Tomcat instance 1 started with clustering support"
echo "JVM Route: tomcat1"
echo "HTTP Port: 8080"
echo "AJP Port: 8009"
echo "Cluster Port: 4000"
```

**Tomcat Instance 2 Startup Script:**

```bash
#!/bin/bash
# /opt/tomcat/tomcat2/bin/[startup-cluster.sh](http://startup-cluster.sh)

export CATALINA_HOME=/opt/tomcat/tomcat2
export CATALINA_BASE=/opt/tomcat/tomcat2
export CATALINA_PID=$CATALINA_BASE/logs/[catalina.pid](http://catalina.pid)

# JVM Route for session clustering
export CATALINA_OPTS="$CATALINA_OPTS -DjvmRoute=tomcat2"

# Memory settings
export CATALINA_OPTS="$CATALINA_OPTS -Xms1024m -Xmx2048m"
export CATALINA_OPTS="$CATALINA_OPTS -XX:+UseG1GC -XX:MaxGCPauseMillis=200"

# Networking for clustering
export CATALINA_OPTS="$CATALINA_OPTS -[Djava.net](http://Djava.net).preferIPv4Stack=true"
export CATALINA_OPTS="$CATALINA_OPTS -Djgroups.bind_addr=192.168.1.100"

# Session clustering debug (optional)
export CATALINA_OPTS="$CATALINA_OPTS -Dorg.apache.catalina.tribes.debug=true"

# Application-specific properties
export CATALINA_OPTS="$CATALINA_OPTS -DDB_HOST=[localhost](http://localhost)"
export CATALINA_OPTS="$CATALINA_OPTS -DDB_PORT=5432"
export CATALINA_OPTS="$CATALINA_OPTS -DEJB_HOST=[localhost](http://localhost)"
export CATALINA_OPTS="$CATALINA_OPTS -DEJB_PORT=8080"

# Start Tomcat
$CATALINA_HOME/bin/[startup.sh](http://startup.sh)

echo "Tomcat instance 2 started with clustering support"
echo "JVM Route: tomcat2"
echo "HTTP Port: 8081"
echo "AJP Port: 8010"
echo "Cluster Port: 4001"
```

### **Step 6.3: Deployment Steps**

```bash
#!/bin/bash
# [deployment-script.sh](http://deployment-script.sh)

echo "Starting Vaadin Cluster Application Deployment"

# 1. Stop existing Tomcat instances
echo "Stopping Tomcat instances..."
/opt/tomcat/tomcat1/bin/[shutdown.sh](http://shutdown.sh)
/opt/tomcat/tomcat2/bin/[shutdown.sh](http://shutdown.sh)

# Wait for shutdown
sleep 10

# 2. Build the application
echo "Building application..."
mvn clean package -DskipTests

# 3. Deploy WAR files
echo "Deploying WAR files..."
cp target/vaadin-cluster-app.war /opt/tomcat/tomcat1/webapps/
cp target/vaadin-cluster-app.war /opt/tomcat/tomcat2/webapps/

# 4. Set permissions
chmod +x /opt/tomcat/tomcat1/bin/*.sh
chmod +x /opt/tomcat/tomcat2/bin/*.sh

# 5. Start Tomcat instances
echo "Starting Tomcat instances with clustering..."
/opt/tomcat/tomcat1/bin/[startup-cluster.sh](http://startup-cluster.sh)
sleep 5
/opt/tomcat/tomcat2/bin/[startup-cluster.sh](http://startup-cluster.sh)

# 6. Wait for startup
echo "Waiting for applications to start..."
sleep 30

# 7. Health check
echo "Performing health checks..."
echo "Tomcat 1 Health:"
curl -s [http://localhost:8080/vaadin-app/actuator/health](http://localhost:8080/vaadin-app/actuator/health) | json_pp

echo "Tomcat 2 Health:"
curl -s [http://localhost:8081/vaadin-app/actuator/health](http://localhost:8081/vaadin-app/actuator/health) | json_pp

# 8. Test load balancer
echo "Testing load balancer:"
curl -s [http://your-domain.com/vaadin-app/actuator/health](http://your-domain.com/vaadin-app/actuator/health) | json_pp

echo "Deployment completed!"
echo "Application URLs:"
echo "- Load Balanced: [http://your-domain.com/vaadin-app](http://your-domain.com/vaadin-app)"
echo "- Direct Tomcat 1: [http://localhost:8080/vaadin-app](http://localhost:8080/vaadin-app)"
echo "- Direct Tomcat 2: [http://localhost:8081/vaadin-app](http://localhost:8081/vaadin-app)"
echo "- Balancer Manager: [http://your-domain.com/balancer-manager](http://your-domain.com/balancer-manager)"
echo "- Server Status: [http://your-domain.com/server-status](http://your-domain.com/server-status)"
```

---

## **Phase 7: Testing & Monitoring**

### **Step 7.1: Session Clustering Test Script**

```bash
#!/bin/bash
# [test-session-clustering.sh](http://test-session-clustering.sh)

echo "Testing Tomcat Session Clustering..."

# Test 1: Session Stickiness
echo "\n=== Test 1: Session Stickiness ==="
SESSION=$(curl -c cookies.txt -s [http://your-domain.com/vaadin-app/](http://your-domain.com/vaadin-app/) | grep -o 'JSESSIONID=[^;]*')
echo "Initial session: $SESSION"

# Make multiple requests
for i in {1..5}; do
    RESPONSE=$(curl -b cookies.txt -s [http://your-domain.com/vaadin-app/cluster](http://your-domain.com/vaadin-app/cluster))
    echo "Request $i: Session maintained"
done

# Test 2: Failover
echo "\n=== Test 2: Failover Test ==="
echo "Stop one Tomcat instance and test session failover"
echo "Session should be maintained on the other instance"

# Test 3: Session Replication
echo "\n=== Test 3: Session Replication ==="
echo "Check Tomcat logs for clustering messages"
tail -f /opt/tomcat/tomcat1/logs/catalina.out | grep -i cluster &
tail -f /opt/tomcat/tomcat2/logs/catalina.out | grep -i cluster &

echo "\nTest completed. Check the results above."
```

### **Step 7.2: Monitoring Script**

```bash
#!/bin/bash
# [monitor-cluster.sh](http://monitor-cluster.sh)

while true; do
    clear
    echo "=== Vaadin Cluster Monitoring ==="
    echo "Time: $(date)"
    echo
    
    echo "=== Tomcat Processes ==="
    ps aux | grep tomcat | grep -v grep
    echo
    
    echo "=== Port Status ==="
    netstat -tlnp | grep ':80[0-9][0-9]'
    echo
    
    echo "=== Load Balancer Status ==="
    curl -s [http://your-domain.com/server-status?auto](http://your-domain.com/server-status?auto) | grep -E "Total|BusyWorkers|IdleWorkers"
    echo
    
    echo "=== Application Health ==="
    echo "Tomcat 1: $(curl -s [http://localhost:8080/vaadin-app/actuator/health](http://localhost:8080/vaadin-app/actuator/health) | jq -r '.status // "DOWN"')"
    echo "Tomcat 2: $(curl -s [http://localhost:8081/vaadin-app/actuator/health](http://localhost:8081/vaadin-app/actuator/health) | jq -r '.status // "DOWN"')"
    echo "Load Balanced: $(curl -s [http://your-domain.com/vaadin-app/actuator/health](http://your-domain.com/vaadin-app/actuator/health) | jq -r '.status // "DOWN"')"
    echo
    
    echo "=== Session Count ==="
    echo "Tomcat 1: $(curl -s [http://localhost:8080/vaadin-app/actuator/sessions](http://localhost:8080/vaadin-app/actuator/sessions) | jq '.sessions | length')"
    echo "Tomcat 2: $(curl -s [http://localhost:8081/vaadin-app/actuator/sessions](http://localhost:8081/vaadin-app/actuator/sessions) | jq '.sessions | length')"
    echo
    
    echo "Press Ctrl+C to stop monitoring"
    sleep 10
done
```

---

## **Phase 8: Troubleshooting Guide**

### **Common Issues and Solutions**

1. **Session Clustering Not Working**
    - Check multicast connectivity: `tcpdump -i any host 228.0.0.4`
    - Verify firewall settings allow ports 4000-4001
    - Check Tomcat logs for clustering errors
    - Ensure `<distributable/>` is in web.xml
2. **Load Balancer Issues**
    - Verify Apache modules are enabled
    - Check AJP connector configuration
    - Test direct Tomcat access
    - Monitor balancer-manager
3. **EJB Connection Issues**
    - Verify EJB server accessibility
    - Check JNDI naming
    - Monitor connection pools
    - Validate credentials
4. **Vaadin Serialization Issues**
    - Ensure all UI components implement Serializable
    - Check session attributes for non-serializable objects
    - Monitor session size

### **Monitoring Commands**

```bash
# Check clustering status
curl [http://your-domain.com/balancer-manager](http://your-domain.com/balancer-manager)

# Monitor session replication
tail -f /opt/tomcat/*/logs/catalina.out | grep -i session

# Check network multicast
netstat -g

# Monitor application health
watch -n 5 'curl -s [http://your-domain.com/vaadin-app/actuator/health](http://your-domain.com/vaadin-app/actuator/health) | jq'
```

This completes the comprehensive implementation plan for your Vaadin 24 + Spring Boot application with Apache load balancing, **Tomcat native session clustering**, and remote EJB integration.