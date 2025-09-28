# Spring Boot 3.4.5 Security Headers — Summary

### What to add and why

- Strict-Transport-Security (HSTS): Enforces HTTPS to prevent downgrade and cookie theft.
- Content-Security-Policy (CSP): Primary defense against XSS and many injections by allow‑listing sources.
- X-Content-Type-Options: Stops MIME type sniffing of responses.
- X-Frame-Options / frame-ancestors: Prevents clickjacking by disallowing iframe embedding.
- Referrer-Policy: Reduces outbound data leakage.
- Permissions-Policy: Disables powerful browser features by default.
- Cross-Origin-Opener-Policy, Cross-Origin-Embedder-Policy, Cross-Origin-Resource-Policy: Context isolation and resource controls to reduce cross‑origin risks.
- Cache-Control (no-store for sensitive): Prevents caching sensitive responses.
- Remove Server/X-Powered-By: Reduces server/framework fingerprinting.

---

### Spring Security configuration (centralized headers)

```java
@Configuration
@EnableWebSecurity
public class SecurityHeadersConfig {
  @Bean
  SecurityFilterChain security(HttpSecurity http) throws Exception {
    return http
      .headers(h -> h
        .httpStrictTransportSecurity(hsts -> hsts.includeSubDomains(true).preload(true).maxAgeInSeconds(31536000))
        .contentSecurityPolicy(csp -> csp.policyDirectives(
          "default-src 'self'; " +
          "script-src 'self' 'strict-dynamic' 'nonce-{nonce}'; " +
          "object-src 'none'; " +
          "style-src 'self'; " +
          "img-src 'self' data:; " +
          "connect-src 'self'; " +
          "frame-ancestors 'none'; " +
          "base-uri 'self'; " +
          "form-action 'self'"))
        .contentTypeOptions(c -> {})
        .frameOptions(f -> f.deny())
        .referrerPolicy(r -> r.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
        .permissionsPolicy(p -> p.policy("camera=(), microphone=(), geolocation=(), payment=(), usb=(), fullscreen=(self)"))
        .crossOriginOpenerPolicy(coop -> coop.policy(CrossOriginOpenerPolicyHeaderWriter.SameOrigin.POLICY))
        .crossOriginEmbedderPolicy(coep -> coep.policy("require-corp"))
        .crossOriginResourcePolicy(corp -> corp.policy("same-origin"))
      )
      .build();
  }
}
```

---

### Per-request CSP nonce

```java
@Component
public class CspNonceFilter extends OncePerRequestFilter {
  public static final String NONCE_ATTR = "cspNonce";
  @Override
  protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
      throws ServletException, IOException {
    String nonce = Base64.getEncoder().encodeToString(SecureRandom.getInstanceStrong().generateSeed(16));
    req.setAttribute(NONCE_ATTR, nonce);
    String csp = res.getHeader("Content-Security-Policy");
    if (csp != null) res.setHeader("Content-Security-Policy", csp.replace("{nonce}", nonce));
    chain.doFilter(req, res);
  }
}
```

- In views, use: <script nonce="${requestScope.cspNonce}"> for any necessary inline bootstrap.

---

### Cache-control for sensitive responses

```java
@Configuration
public class CacheControlConfig implements WebMvcConfigurer {
  @Override
  public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(new HandlerInterceptor() {
      @Override
      public void postHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler, ModelAndView modelAndView) {
        response.setHeader("Cache-Control", "no-store");
      }
    });
  }
}
```

---

### Reduce server fingerprinting (Tomcat)

```java
@Configuration
public class TomcatHeaderHidingConfig {
  @Bean
  WebServerFactoryCustomizer<TomcatServletWebServerFactory> customizeConnector() {
    return factory -> factory.addConnectorCustomizers(connector -> connector.setXpoweredBy(false));
  }
}
```

---

### Rollout plan

1. Enable low‑risk headers immediately: X-Content-Type-Options, X-Frame-Options or frame-ancestors, Referrer-Policy, Permissions-Policy, Cache-Control for sensitive endpoints, remove Server banner.
2. Turn on HSTS without preload. Monitor two weeks, then add preload if all subdomains are HTTPS.
3. Start CSP in Report‑Only. Collect and fix violations. Move inline JS to nonce‑based or external.
4. Enforce CSP. Keep a secondary Report‑Only for drift detection.
5. Add COOP/COEP/CORP if compatible with embeds.

---

### Validation checklist

- Security headers present on all routes, including errors and redirects
- CSP has no 'unsafe-inline' or 'unsafe-eval'; uses nonces or hashes
- Actuator and static assets covered appropriately
- CORS and CSRF configured explicitly and minimally
- Re-scan with a header scanner after each change

---

### Practical prevention examples

### 1) Cross‑Site Scripting (XSS) → Prevented by CSP

- Problem: An attacker injects `<script>alert('pwned')</script>` into a comment field.
- Without CSP: The script executes in the victim’s browser.
- With CSP: `script-src 'self' 'strict-dynamic' 'nonce-{nonce}'; object-src 'none'`
    - Only scripts from your origin and with your nonce run. Inline scripts without the nonce are blocked.
- Demo test:

```bash
# Response should include a CSP header and block inline script
curl -I [https://yourapp.example/app](https://yourapp.example/app)
```

- Fix pattern in views:

```html
<script nonce="${requestScope.cspNonce}">
  // small bootstrap that reads data-* attributes and loads external JS from 'self'
</script>
```

### 2) Clickjacking → Prevented by frame-ancestors or X-Frame-Options

- Problem: Attacker iframes your app and overlays fake UI to trick clicks.
- Header:

```
Content-Security-Policy: frame-ancestors 'none'
# or legacy fallback
X-Frame-Options: DENY
```

- Verification:

```bash
curl -I [https://yourapp.example](https://yourapp.example) | grep -E "(Content-Security-Policy|X-Frame-Options)"
```

### 3) MIME sniffing → Prevented by X-Content-Type-Options

- Problem: Browser “sniffs” a text file as HTML/JS and executes it.
- Header:

```
X-Content-Type-Options: nosniff
```

- Also set Content-Type correctly on responses and static files.

### 4) Downgrade and mixed content → Prevented by HSTS

- Problem: Users click http:// links or are forced to HTTP by a network attacker.
- Header:

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

- Roll out without `preload` first. Ensure all subdomains are HTTPS before adding `preload`.

### 5) Referrer leakage → Controlled with Referrer-Policy

- Problem: Full URLs with tokens/IDs leak to third‑party sites via the Referer header.
- Header:

```
Referrer-Policy: strict-origin-when-cross-origin
```

- Behavior: Sends only the origin to cross‑site targets, full URL on same‑origin.

### 6) Powerful browser features → Restricted with Permissions-Policy

- Problem: Rogue third‑party content prompts for camera, mic, or location.
- Header:

```
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), usb=(), fullscreen=(self)
```

### 7) Cross‑origin data exfiltration → CORP/COEP/COOP isolation

- Problem: Cross‑origin scripts or windows interact in ways that leak data.
- Headers:

```
Cross-Origin-Resource-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Opener-Policy: same-origin
```

- Note: Test thoroughly; these can block third‑party embeds unless they send proper CORS or CORP headers.

### 8) Sensitive responses cached → Prevented by Cache-Control

- Problem: Authenticated API responses stored in browser/shared caches.
- Header:

```
Cache-Control: no-store
```

- Apply to authenticated pages and API responses returning sensitive data.

---

### End‑to‑end example: Secure Spring Boot headers

Expected response headers for an authenticated page (simplified):

---

### Stack‑specific details and deeper examples

### A) Vaadin 24 + Spring Boot

CSP adjustments commonly needed by Vaadin apps:

- script-src: include 'self' and a nonce; if using Vaadin dev tools or dynamic imports, avoid 'unsafe-eval' and prefer nonce. For Vaadin Push over WebSocket, add the origin to connect-src.
- style-src: allow 'self' and optionally 'unsafe-inline' only if absolutely necessary; prefer hashed inline styles or avoid inline styles.
- img-src: 'self' data: blob: if you render charts or images via blobs.
- font-src: 'self' data: or your CDN.
- connect-src: 'self' wss://your-domain for Push.

Example CSP tuned for production Vaadin:

```
Content-Security-Policy: 
 default-src 'self'; 
 script-src 'self' 'strict-dynamic' 'nonce-<nonce>'; 
 style-src 'self'; 
 img-src 'self' data: blob:; 
 connect-src 'self' wss://[app.example.com](http://app.example.com); 
 font-src 'self' data:; 
 frame-ancestors 'none'; 
 base-uri 'self'; 
 form-action 'self'
```

Vaadin bootstrap inline script:

```html
<!-- In index.html or a Vaadin template -->
<script nonce="${requestScope.cspNonce}">
  window.__vaadinNonce = '${requestScope.cspNonce}';
  // Any minimal bootstrap logic goes here
</script>
```

### B) Thymeleaf templates with CSP nonce

Add the nonce to inline bootstraps and prefer external JS for app logic.

```html
<!DOCTYPE html>
<html xmlns:th="[http://www.thymeleaf.org](http://www.thymeleaf.org)">
<head>
  <meta charset="utf-8"/>
  <title>App</title>
</head>
<body>
  <div id="root" th:data-user="${user}"></div>
  <script nonce="${request.cspNonce}" th:inline="javascript">
    const user = /*[[${user}]]*/ null;
  </script>
  <script src="/static/app.js"></script>
</body>
</html>
```

### C) Apache HTTP Server edge config (front of Spring Boot)

```
# Enable module
LoadModule headers_module modules/mod_[headers.so](http://headers.so)

# Security headers
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
Header always set Content-Security-Policy "default-src 'self'; frame-ancestors 'none'; base-uri 'self'"
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "DENY"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
Header always set Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), fullscreen=(self)"
# Remove banners
Header unset Server
Header always unset X-Powered-By
```

### D) Nginx edge config

```
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header Content-Security-Policy "default-src 'self'; frame-ancestors 'none'; base-uri 'self'" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), fullscreen=(self)" always;
# Hide server tokens
server_tokens off;
```

### E) Actuator hardening

- Expose only required endpoints and secure them.
- Ensure headers are applied to actuator responses as well.

```yaml
# application.yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: when_authorized
```

Spring Security matchers example:

```java
http
  .securityMatcher(new AntPathRequestMatcher("/actuator/**"))
  .authorizeHttpRequests(a -> a.requestMatchers("/actuator/health").permitAll()
                           .anyRequest().hasRole("OPS"))
  .headers(h -> h /* same headers as main chain, or apply globally */);
```

### F) Automated tests for headers (MockMvc)

```java
@SpringBootTest
@AutoConfigureMockMvc
class SecurityHeadersIT {
  @Autowired MockMvc mvc;

  @Test void headers_present_on_home() throws Exception {
    mvc.perform(get("/").secure(true))
      .andExpect(header().string("Strict-Transport-Security", Matchers.containsString("max-age=")))
      .andExpect(header().string("Content-Security-Policy", Matchers.containsString("default-src 'self'")))
      .andExpect(header().string("X-Content-Type-Options", "nosniff"))
      .andExpect(header().string("Referrer-Policy", "strict-origin-when-cross-origin"));
  }
}
```

### G) Common pitfalls and fixes

- CSP breaks app due to inline scripts
    - Fix: move inline JS to external files or apply a per-request nonce and add 'nonce-<nonce>' to script-src.
- Using 'unsafe-inline' or 'unsafe-eval'
    - Avoid these; they weaken XSS protection. If unavoidable in the short term, restrict scope and plan removal.
- Third‑party CDNs blocked by CSP
    - Add exact domains to script-src/style-src/img-src as needed. Prefer subresource integrity and minimal third parties.
- COEP/COOP/Corp break embeds
    - Either remove for pages that need embeds or ensure the third party serves appropriate CORS/CORP headers.
- HSTS preload too early
    - Preload only after all subdomains are permanently HTTPS.

### H) Expanded curl verification

```bash
URL="[https://your-app.example](https://your-app.example)"
# Show all relevant security headers
curl -sI "$URL" | egrep -i "^(strict-transport-security|content-security-policy|x-content-type-options|x-frame-options|referrer-policy|permissions-policy|cross-origin-.*|cache-control):"
# Ensure CSP blocks inline script (should see violation in DevTools when testing a page with injected inline script)
```

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'; script-src 'self' 'strict-dynamic' 'nonce-<nonce>'; object-src 'none'; style-src 'self'; img-src 'self' data:; connect-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), usb=(), fullscreen=(self)
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Resource-Policy: same-origin
Cache-Control: no-store
```

Quick verification script:

---

### Appendix: Complete, copy‑pasteable examples

### 1) Curl verification with a real URL

```bash
URL="[https://yourapp.example](https://yourapp.example)"
# Show key headers
curl -sI "$URL" | egrep -i "^(strict-transport-security|content-security-policy|x-content-type-options|x-frame-options|referrer-policy|permissions-policy|cross-origin-.*|cache-control):"

# Expect DENY or frame-ancestors 'none'
curl -sI "$URL" | egrep -i "^(x-frame-options|content-security-policy)"
```

### 2) Vaadin 24 production CSP tuned

```
Content-Security-Policy:
 default-src 'self';
 script-src 'self' 'strict-dynamic' 'nonce-<nonce>';
 style-src 'self';
 img-src 'self' data: blob:;
 connect-src 'self' wss://yourapp.example;
 font-src 'self' data:;
 frame-ancestors 'none';
 base-uri 'self';
 form-action 'self'
```

### 3) Thymeleaf template with nonce

```html
<!DOCTYPE html>
<html xmlns:th="[http://www.thymeleaf.org](http://www.thymeleaf.org)">
<head>
  <meta charset="utf-8"/>
  <title>App</title>
</head>
<body>
  <div id="root" th:data-user="${user}"></div>
  <script nonce="${request.cspNonce}" th:inline="javascript">
    const user = /*[[${user}]]*/ null;
  </script>
  <script src="/static/app.js"></script>
</body>
</html>
```

### 4) Apache complete header block

```
# In VirtualHost or main config
LoadModule headers_module modules/mod_[headers.so](http://headers.so)

Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
Header always set Content-Security-Policy "default-src 'self'; frame-ancestors 'none'; base-uri 'self'"
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "DENY"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
Header always set Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), fullscreen=(self)"
Header always unset X-Powered-By
Header unset Server
```

### 5) Nginx complete header block

```
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header Content-Security-Policy "default-src 'self'; frame-ancestors 'none'; base-uri 'self'" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), fullscreen=(self)" always;
server_tokens off;
```

### 6) MockMvc test: verify all headers

```java
@SpringBootTest
@AutoConfigureMockMvc
class SecurityHeadersIT {
  @Autowired MockMvc mvc;

  @Test void headers_present_on_home() throws Exception {
    mvc.perform(get("/").secure(true))
      .andExpect(header().string("Strict-Transport-Security", Matchers.containsString("max-age=")))
      .andExpect(header().string("Content-Security-Policy", Matchers.containsString("default-src 'self'")))
      .andExpect(header().string("X-Content-Type-Options", "nosniff"))
      .andExpect(header().string("Referrer-Policy", "strict-origin-when-cross-origin"))
      .andExpect(header().string("Permissions-Policy", Matchers.containsString("camera=()")))
      .andExpect(header().string("Cross-Origin-Opener-Policy", "same-origin"));
  }
}
```

```bash
url=[https://yourapp.example](https://yourapp.example)
curl -sI "$url" | egrep -i "^(strict-transport-security|content-security-policy|x-content-type-options|x-frame-options|referrer-policy|permissions-policy|cross-origin-.*|cache-control):"
```