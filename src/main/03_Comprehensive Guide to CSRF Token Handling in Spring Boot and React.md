# Comprehensive Guide to CSRF Token Handling in Spring Boot and React

## Introduction to CSRF

Cross-Site Request Forgery (CSRF) is a type of malicious exploit where unauthorized commands are transmitted from a user that the web application trusts. This attack tricks the victim into submitting a malicious request. It exploits the trust that a web application has in an authenticated user. Unlike Cross-Site Scripting (XSS), which exploits the trust a user has in a particular site, CSRF exploits the trust a site has in a user's browser [1].

CSRF attacks typically involve an attacker crafting a malicious request and tricking an authenticated user into unknowingly executing it. Since the user is already authenticated with the legitimate website, their browser automatically includes any session cookies or other credentials with the malicious request. The server, unable to distinguish between a legitimate request and a forged one, processes the malicious request as if it were initiated by the legitimate user [2].

The format of a CSRF attack follows this pattern:

**Malicious Site → User’s Browser → Legitimate Site (with user’s cookies)** [2]

To prevent this, CSRF protection involves issuing a unique token with each user session and requiring this token on all state-changing requests. This ensures that only legitimate requests from your frontend can be validated by the server [2].

## Spring Boot CSRF Configuration

Spring Security provides robust CSRF protection by default for unsafe HTTP methods (e.g., POST, PUT, DELETE). When CSRF protection is enabled, Spring Security expects a CSRF token to be present in requests that modify the application's state. If the token is missing or invalid, the request is rejected [1].

### Default Configuration

In Spring Boot 3.x with Spring Security 6.x, enabling CSRF protection is straightforward. By default, Spring Security uses `HttpSessionCsrfTokenRepository`, which stores the CSRF token in the `HttpSession`. You can enable it in your `SecurityFilterChain` configuration:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

	@Bean
	public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
		http
			// ... other configurations
			.csrf(Customizer.withDefaults());
		return http.build();
	}
}
```

### Using CookieCsrfTokenRepository for JavaScript Applications

For single-page applications (SPAs) built with JavaScript frameworks like React, it is often more convenient to store the CSRF token in a cookie rather than the session. This allows the frontend application to easily retrieve the token from the cookie and include it in subsequent requests. Spring Security provides `CookieCsrfTokenRepository` for this purpose [1].

When `CookieCsrfTokenRepository` is used, Spring Security will send the CSRF token in a cookie, typically named `XSRF-TOKEN`. The JavaScript application can then read this cookie and send its value as a header (e.g., `X-XSRF-TOKEN`) in subsequent requests [1].

To configure `CookieCsrfTokenRepository` in your Spring Boot application:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

	@Bean
	public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
		http
			// ... other configurations
			.csrf((csrf) -> csrf
				.csrfTokenRepository(CookieCsrfTokenRepository.withDefaults()))
			;
		return http.build();
	}
}
```

This configuration will instruct Spring Security to manage the CSRF token via a cookie. The token will be available to your React application to be included in requests.

## React CSRF Token Integration

When working with a React frontend and a Spring Boot backend with CSRF protection enabled, the React application needs to perform two main steps:

1.  **Retrieve the CSRF token:** The token is typically sent by the server in a cookie (if `CookieCsrfTokenRepository` is used) or as part of the initial page load (e.g., in a meta tag or a dedicated endpoint).
2.  **Include the CSRF token in requests:** For all state-changing requests (POST, PUT, DELETE), the React application must include the retrieved CSRF token in a specific header that the Spring Boot backend expects.

### Retrieving the CSRF Token

If you've configured Spring Boot to use `CookieCsrfTokenRepository`, the CSRF token will be available in a cookie named `XSRF-TOKEN`. You can access this cookie in your React application. A common approach is to use a utility function to read cookie values.

### Including the CSRF Token in Requests with Axios

Axios is a popular HTTP client for JavaScript that makes it easy to send requests. It can be configured to automatically include the CSRF token in request headers. Spring Security, by default, expects the CSRF token in the `X-XSRF-TOKEN` header when using `CookieCsrfTokenRepository` [1].

Here's an example of how you can set up Axios to handle CSRF tokens in your React application:

```javascript
import axios from 'axios';

// Function to get a cookie value by name
function getCookie(name) {
  const value = `; ${document.cookie}`;
  const parts = value.split(`; ${name}=`);
  if (parts.length === 2) return parts.pop().split(';').shift();
}

// Set up Axios to include the CSRF token in requests
axios.interceptors.request.use(config => {
  const csrfToken = getCookie('XSRF-TOKEN');
  if (csrfToken) {
    config.headers['X-XSRF-TOKEN'] = csrfToken;
  }
  return config;
}, error => {
  return Promise.reject(error);
});

// Example POST request using Axios
axios.post('/api/data', { foo: 'bar' })
  .then(response => {
    console.log('Data sent successfully:', response.data);
  })
  .catch(error => {
    console.error('Error sending data:', error);
  });

// Example GET request (CSRF token not typically required for GET)
axios.get('/api/user')
  .then(response => {
    console.log('User data:', response.data);
  })
  .catch(error => {
    console.error('Error fetching user data:', error);
  });
```

In this example:

*   The `getCookie` function reads the `XSRF-TOKEN` cookie.
*   An Axios interceptor is used to automatically add the `X-XSRF-TOKEN` header to every outgoing request if the `XSRF-TOKEN` cookie is present. This ensures that all your POST, PUT, and DELETE requests will include the necessary CSRF token.

### Important Considerations for React and CSRF

*   **Automatic Token Inclusion:** React itself does not automatically add CSRF tokens to backend requests. You need to explicitly retrieve the token from the server (e.g., from a cookie or a meta tag) and include it in your HTTP requests, typically via a library like Axios or Fetch API [2].
*   **SameSite Cookies:** Modern browsers implement `SameSite` cookie attributes (`Strict`, `Lax`, `None`) to provide some level of CSRF protection. `SameSite=Lax` (the default for many browsers) prevents cookies from being sent with cross-site requests initiated by third-party sites, except for top-level navigations with safe HTTP methods (GET). While helpful, `SameSite` cookies alone are not a complete defense against CSRF and should be used in conjunction with CSRF tokens [2].
*   **CORS vs. CSRF:** It's a common misconception that Cross-Origin Resource Sharing (CORS) prevents CSRF attacks. CORS is designed to prevent data leaks across origins, not to stop unauthorized actions. CSRF attacks can still succeed even when CORS is configured, as CORS controls reading responses, not sending requests [2].

## Best Practices for CSRF Protection

To ensure robust CSRF protection in your Spring Boot and React application, consider the following best practices:

1.  **Always Enable CSRF Protection:** Do not disable CSRF protection unless absolutely necessary and you fully understand the implications. It's a critical security measure [1].
2.  **Use `CookieCsrfTokenRepository` for SPAs:** For React applications, storing the CSRF token in an `HttpOnly` cookie (which `CookieCsrfTokenRepository` does by default) and then reading it with JavaScript to include in a custom header is a widely accepted and secure pattern [1].
3.  **Send Token in Custom Header:** Always send the CSRF token in a custom HTTP header (e.g., `X-XSRF-TOKEN`) rather than as a request parameter. This helps prevent the token from being leaked in server logs or browser history [1].
4.  **Protect All State-Changing Operations:** Ensure that all HTTP methods that modify data on the server (POST, PUT, DELETE, PATCH) are protected by CSRF. GET requests should ideally be idempotent and not change server state, thus not requiring CSRF protection [1].
5.  **Secure Your Cookies:** Ensure your session cookies and CSRF token cookies are marked as `Secure` (sent only over HTTPS) and `HttpOnly` (not accessible via client-side JavaScript, though `XSRF-TOKEN` needs to be readable by JS, so `HttpOnly` might not be applicable for that specific cookie if you're reading it via `document.cookie`). The `XSRF-TOKEN` cookie typically does not need to be `HttpOnly` as it's designed to be read by JavaScript [1].
6.  **Implement `SameSite` Cookies:** Configure `SameSite=Lax` or `SameSite=Strict` for your session cookies. While not a standalone solution, it adds an extra layer of defense [2].
7.  **Avoid Disabling CSRF in Development:** While tempting for convenience, disabling CSRF in development can lead to vulnerabilities being overlooked. Instead, use proper development practices that respect security, such as using test tokens or configuring your development environment to correctly handle CSRF [2].
8.  **Handle Token Expiration:** Be aware that CSRF tokens are typically tied to the user's session. If the session expires, the token becomes invalid. Your frontend should handle cases where the server rejects a request due to an invalid or expired CSRF token (e.g., by redirecting to a login page or prompting for re-authentication) [1].

## Conclusion

Implementing CSRF protection is a crucial step in securing your web application. By correctly configuring Spring Boot to issue and validate CSRF tokens, and by ensuring your React application retrieves and includes these tokens in its requests, you can effectively mitigate the risk of CSRF attacks. Following best practices, such as using `CookieCsrfTokenRepository` and sending tokens in custom headers, will lead to a robust and secure full-stack application.

## References

[1] Cross Site Request Forgery (CSRF) :: Spring Security. Available at: https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html
[2] React CSRF Protection Guide: Examples and How to Enable It. Available at: https://www.stackhawk.com/blog/react-csrf-protection-guide-examples-and-how-to-enable-it/




## Comparison: Vaadin vs. Spring Boot/React CSRF Handling

The fundamental goal of CSRF protection remains the same across different web technologies: to ensure that state-changing requests originate from the legitimate application and not from malicious third-party sites. However, the implementation and developer interaction with CSRF protection differ significantly between Vaadin and a traditional Spring Boot backend with a React frontend.

### Vaadin's Approach (Server-Side UI Framework)

Vaadin is a server-side UI framework, meaning that the UI logic and component state primarily reside on the server. The client-side (browser) acts as a thin client, sending user interactions to the server and rendering updates received from the server. This architecture inherently simplifies CSRF protection for developers [3].

*   **Automatic Handling:** Vaadin automatically handles CSRF token generation, inclusion, and validation. All requests between the client and the server, which are managed by Vaadin's communication layer, include a user session-specific CSRF token. Developers typically do not need to manually retrieve or include CSRF tokens in their code [3].
*   **Token Location:** The CSRF token is passed within the JSON message in the request body for Vaadin Flow applications [3]. For Hilla (Vaadin 24), the CSRF token is delivered as part of the initial bootstrap HTML when the application is first loaded, and then included in each endpoint call [4].
*   **Integrated Security:** Vaadin's security mechanisms are deeply integrated into the framework. This means that as long as you are using Vaadin's standard communication channels, CSRF protection is largely taken care of for you. This reduces the burden on developers and minimizes the chance of misconfiguration [3].
*   **Less Developer Overhead:** Because Vaadin manages the client-server communication and CSRF tokens internally, developers do not need to write client-side JavaScript code to fetch tokens from cookies or inject them into request headers, as is common with RESTful APIs and separate frontends.

### Spring Boot/React Approach (Separate Backend and Frontend)

In a Spring Boot backend with a React frontend, the backend (Spring Boot) provides a RESTful API, and the frontend (React) consumes this API. This architecture offers greater flexibility but places more responsibility on the developer for security, including CSRF protection.

*   **Manual Token Handling (Client-Side):** React, being a client-side library, does not inherently know about CSRF tokens generated by the backend. Developers must explicitly retrieve the CSRF token from the Spring Boot backend (e.g., from a cookie or a dedicated endpoint) and then manually include it in state-changing requests (POST, PUT, DELETE) from the React application [1, 2].
*   **Token Location:** Spring Boot, when configured with `CookieCsrfTokenRepository`, typically sends the CSRF token in an `XSRF-TOKEN` cookie. The React application then reads this cookie and sends the token back in a custom HTTP header, commonly `X-XSRF-TOKEN` [1].
*   **Axios/Fetch Configuration:** Developers often use HTTP clients like Axios or the Fetch API in React to make requests to the backend. These clients need to be configured (e.g., using interceptors in Axios) to automatically attach the retrieved CSRF token to relevant outgoing requests [2].
*   **Increased Developer Responsibility:** The developer is responsible for ensuring that the CSRF token is correctly retrieved, stored (even if temporarily in memory), and included in all necessary requests. Any oversight in this process can lead to CSRF vulnerabilities.

### Key Differences Summarized

| Feature                | Vaadin (Server-Side UI)                                  | Spring Boot/React (Separate Frontend/Backend)                               |
| :--------------------- | :------------------------------------------------------- | :-------------------------------------------------------------------------- |
| **CSRF Handling**      | Automatic, handled by the framework                      | Manual, requires developer implementation on client-side                    |
| **Token Retrieval**    | Internal to framework, token passed in JSON message/bootstrap HTML | Client-side JavaScript reads from cookie or dedicated endpoint              |
| **Token Inclusion**    | Automatic, within Vaadin's communication                 | Manual, requires configuring HTTP client (e.g., Axios interceptors)         |
| **Developer Overhead** | Low                                                      | Higher                                                                      |
| **Architecture**       | Server-side UI rendering                                 | Client-side UI rendering with RESTful API backend                           |

In essence, Vaadin's server-side nature allows it to abstract away much of the complexity of CSRF protection, providing a more 


streamlined security experience. In contrast, the separate frontend/backend approach of Spring Boot and React requires more explicit developer involvement in managing CSRF tokens.

## Best Practices for CSRF Protection in Vaadin 24

While Vaadin handles much of the CSRF protection automatically, there are still best practices to consider to ensure the highest level of security for your Vaadin 24 applications:

1.  **Trust Vaadin's Built-in Protection:** For most standard Vaadin applications, the framework's default CSRF protection is robust and sufficient. Avoid trying to implement your own CSRF mechanisms unless you have a very specific and well-understood reason to do so. Vaadin's internal handling is designed to be secure and efficient [3].
2.  **Keep Vaadin Up-to-Date:** Regularly update your Vaadin dependencies to the latest stable versions. Security vulnerabilities are continuously discovered and patched, and keeping your framework updated ensures you benefit from the latest security fixes and improvements [4].
3.  **Do Not Disable CSRF Protection in Production:** Vaadin provides options to disable CSRF protection (e.g., for specific testing scenarios). However, this should **never** be done in a production environment. Disabling CSRF protection leaves your application vulnerable to CSRF attacks [3].
4.  **Be Mindful of Custom Server-Side Endpoints (Hilla/REST):** If you are using Hilla with custom server-side endpoints (e.g., `@RestController` in Spring Boot alongside Hilla endpoints) or integrating with other REST APIs, ensure that those endpoints are also adequately protected. While Hilla endpoints benefit from Vaadin's built-in CSRF, traditional REST controllers in a Spring Boot application will rely on Spring Security's CSRF mechanisms, as discussed in the previous section [1, 4].
5.  **Secure Your Authentication:** CSRF protection relies on the user being authenticated. Ensure your authentication mechanisms (login forms, session management) are secure. Use strong password policies, multi-factor authentication, and secure session management practices.
6.  **Use HTTPS Everywhere:** Always deploy your Vaadin applications over HTTPS. This encrypts communication between the client and server, protecting against various attacks, including man-in-the-middle attacks that could potentially expose or manipulate CSRF tokens.
7.  **Consider `SameSite` Cookies:** While Vaadin handles CSRF internally, ensuring your session cookies are configured with appropriate `SameSite` attributes (`Lax` or `Strict`) adds another layer of defense at the browser level. This is generally handled by Spring Security if you are using it with Vaadin [2].
8.  **Validate All Input on the Server:** Regardless of CSRF protection, always validate and sanitize all user input on the server side. This protects against a wide range of vulnerabilities, including SQL injection and Cross-Site Scripting (XSS).
9.  **Regular Security Audits and Testing:** Periodically conduct security audits and penetration testing on your Vaadin applications. This can help identify any overlooked vulnerabilities or misconfigurations.

## Conclusion

Vaadin 24, particularly with Hilla, offers a highly integrated and largely automatic approach to CSRF protection, significantly simplifying the developer's task compared to building separate frontend and backend applications. By leveraging Vaadin's built-in security features, keeping the framework updated, and adhering to general web security best practices, developers can build robust and secure applications with confidence.

## References

[1] Cross Site Request Forgery (CSRF) :: Spring Security. Available at: https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html
[2] React CSRF Protection Guide: Examples and How to Enable It. Available at: https://www.stackhawk.com/blog/react-csrf-protection-guide-examples-and-how-to-enable-it/
[3] Security in Vaadin applications | Advanced Topics | Flow | Vaadin Docs. Available at: https://vaadin.com/docs/v14/flow/advanced/framework-security
[4] Introduction to security in Hilla | Vaadin. Available at: https://vaadin.com/docs/latest/hilla/guides/security/intro




## Vaadin's Automatic CSRF Handling and Spring Security Integration

Vaadin's approach to CSRF protection is largely automatic and integrated into its communication mechanisms. This means that for most Vaadin applications, especially those using Spring Boot and Spring Security, you don't need to write explicit code to handle CSRF tokens. The framework takes care of it behind the scenes.

### How Vaadin Handles CSRF Automatically

When a Vaadin application loads in the browser, the server-side framework generates a unique CSRF token for the user's session. This token is then embedded within the initial HTML response that is sent to the client. For Vaadin Flow applications, this token is typically included in the JSON messages exchanged between the client and server. For Hilla applications (Vaadin 24), the token is part of the initial bootstrap HTML and is then automatically included in subsequent endpoint calls [3, 4].

The key aspect is that Vaadin's client-side engine is designed to automatically read this token and include it in all subsequent requests that modify the server-side state. This includes user interactions like button clicks, form submissions, and data updates. The server-side then validates this token with each request. If the token is missing or invalid, the request is rejected, thus preventing CSRF attacks.

This automatic handling significantly reduces the burden on developers, as they don't need to manually fetch tokens from cookies, set headers, or manage token lifecycle in their client-side code, unlike traditional REST API and separate frontend setups.

### Conceptual Example of Token Flow (No Code Required by Developer)

Let's illustrate the conceptual flow of a CSRF token in a Vaadin application:

1.  **Initial Page Load:**
    *   User navigates to `https://your-vaadin-app.com`.
    *   Vaadin server generates a unique CSRF token (e.g., `abc123xyz`).
    *   The server embeds this token in the initial HTML response sent to the browser. For Hilla, this might be a JavaScript variable or a meta tag. For Vaadin Flow, it's part of the initial communication payload.

2.  **User Interaction (e.g., Button Click):**
    *   User clicks a button in the Vaadin UI.
    *   Vaadin's client-side engine automatically includes the `abc123xyz` token in the JSON request body sent to the server.
    *   The request payload sent over XHR (simplified):

    ```json
    {
      "csrfToken": "abc123xyz",
      "rpc": [
        {
          "type": "call",
          "node": "myButton",
          "method": "onClick"
        }
      ],
      "syncId": 1,
      "clientId": 1
    }
    ```

3.  **Server-Side Validation:**
    *   The Vaadin server receives the request.
    *   It extracts the `csrfToken` from the request body.
    *   It compares this token with the expected token stored in the user's session.
    *   If they match, the request is processed. If not, the request is rejected with a 403 Forbidden error.

### Spring Security Integration with Vaadin

When you use Vaadin with Spring Boot and Spring Security, the integration is largely seamless regarding CSRF. Spring Security's default CSRF protection can sometimes conflict with Vaadin's internal mechanisms if not configured correctly. However, Vaadin provides `VaadinWebSecurity` (or similar configurations in older versions) that simplifies this integration.

The best practice is to use `VaadinWebSecurity` (for Vaadin Flow) or ensure that Spring Security's CSRF handling is configured to cooperate with Hilla's built-in mechanisms. Often, this means allowing Vaadin to manage CSRF for its own communication and potentially configuring Spring Security to ignore CSRF for certain public endpoints if necessary, though this is rare for typical Vaadin applications.

Here's a typical Spring Security configuration for a Vaadin application. Notice that no special CSRF configuration is needed beyond the standard Spring Security setup, as Vaadin handles its own CSRF tokens internally:

```java
import com.vaadin.flow.spring.security.VaadinWebSecurity;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@EnableWebSecurity
@Configuration
public class SecurityConfig extends VaadinWebSecurity {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);
        // You can add additional security configurations here if needed
        // For example, to allow access to certain public paths:
        // http.authorizeHttpRequests(auth -> auth.requestMatchers("/public/**").permitAll());
    }

    @Bean
    public UserDetailsService users() {
        UserDetails user = User.withUsername("user")
                .password("{noop}password") // {noop} for plain text password, use encoded in production
                .roles("USER")
                .build();
        UserDetails admin = User.withUsername("admin")
                .password("{noop}admin")
                .roles("ADMIN", "USER")
                .build();
        return new InMemoryUserDetailsManager(user, admin);
    }
}
```

In this configuration:

*   `VaadinWebSecurity` extends Spring Security's `WebSecurityConfigurerAdapter` (or equivalent in newer Spring Security versions) and provides default security configurations suitable for Vaadin applications, including proper CSRF handling for Vaadin's internal communication.
*   You typically override the `configure(HttpSecurity http)` method to add your application-specific security rules, such as defining access control for different URLs or configuring authentication providers.
*   The `UserDetailsService` bean provides in-memory user details for demonstration purposes. In a real application, you would integrate with a database or other identity management system.

**Key Takeaway:** For Vaadin 24, the primary best practice for CSRF is to rely on the framework's built-in mechanisms. Manual intervention for CSRF token handling is rarely needed and can potentially introduce vulnerabilities if not done correctly. Focus on securing your authentication and authorization, and keep your Vaadin and Spring Security dependencies updated.




## Manual Spring Security CSRF Configuration for Vaadin (Without `VaadinWebSecurity`)

While extending `VaadinWebSecurity` is the recommended and simplest approach for integrating Spring Security with Vaadin, there might be scenarios where you choose not to extend it. In such cases, you need to manually configure Spring Security to ensure proper CSRF handling for your Vaadin application. The key challenge here is that Vaadin and Spring Security have their own independent CSRF mechanisms, and they can conflict if not managed carefully [5].

Historically, before `VaadinWebSecurity` (introduced around Vaadin 21), a common approach was to disable Spring Security's CSRF for Vaadin's internal communication paths, allowing Vaadin to handle its own CSRF protection. This is because Vaadin sends its CSRF token in the request body, while Spring Security typically expects it in a header or as a form parameter for its default CSRF protection.

If you don't extend `VaadinWebSecurity`, you will need to configure your `SecurityFilterChain` to:

1.  **Disable Spring Security's CSRF for Vaadin's internal URLs:** Vaadin uses specific internal URLs for its client-server communication (e.g., `/VAADIN/`, `/VAADIN/push`). Requests to these paths should be exempted from Spring Security's CSRF checks, as Vaadin handles them internally.
2.  **Keep Spring Security's CSRF enabled for other application endpoints:** For any other REST endpoints or custom forms that are not part of Vaadin's internal communication, Spring Security's CSRF protection should remain active.

Here's a conceptual example of how you might configure Spring Security manually without extending `VaadinWebSecurity`. This example assumes you are using Spring Security 6.x and focuses on the CSRF aspect. You would still need to configure authentication, authorization, and other security aspects as per your application's requirements.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.csrf.CsrfTokenRequestAttributeHandler;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;
import org.springframework.security.web.csrf.CsrfTokenRepository;
import org.springframework.security.web.util.matcher.AntPathRequestMatcher;

@EnableWebSecurity
@Configuration
public class ManualSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        // Configure CSRF token repository (e.g., using cookies for SPA-like behavior)
        CsrfTokenRepository csrfTokenRepository = CookieCsrfTokenRepository.withDefaults();
        // Set the token request attribute handler to allow token retrieval from request attributes
        CsrfTokenRequestAttributeHandler requestHandler = new CsrfTokenRequestAttributeHandler();

        http
            .csrf(csrf -> csrf
                .csrfTokenRepository(csrfTokenRepository)
                .csrfTokenRequestHandler(requestHandler)
                // Ignore CSRF for Vaadin's internal communication paths
                .ignoringRequestMatchers(
                    new AntPathRequestMatcher("/VAADIN/**"),
                    new AntPathRequestMatcher("/VAADIN/push/**"),
                    new AntPathRequestMatcher("/hilla-dev-mode-gizmo/**"), // For Hilla dev mode
                    new AntPathRequestMatcher("/hilla/**") // For Hilla endpoints
                )
            )
            .authorizeHttpRequests(auth -> auth
                // Allow access to Vaadin internal resources
                .requestMatchers("/VAADIN/**").permitAll()
                .requestMatchers("/VAADIN/push/**").permitAll()
                .requestMatchers("/hilla-dev-mode-gizmo/**").permitAll()
                .requestMatchers("/hilla/**").permitAll()
                // Allow public access to login page and other static resources
                .requestMatchers("/login", "/images/**", "/styles/**").permitAll()
                // All other requests require authentication
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login").permitAll()
                .defaultSuccessUrl("/", true)
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/login?logout")
                .permitAll()
            );

        return http.build();
    }

    // You would typically define your UserDetailsService and PasswordEncoder here
    // For example:
    // @Bean
    // public UserDetailsService userDetailsService() { ... }
    // @Bean
    // public PasswordEncoder passwordEncoder() { ... }
}
```

**Explanation of the Manual Configuration:**

*   **`csrfTokenRepository(CookieCsrfTokenRepository.withDefaults())`**: This configures Spring Security to store and retrieve the CSRF token from a cookie. This is often preferred for modern web applications where the frontend might be a separate SPA.
*   **`csrfTokenRequestHandler(requestHandler)`**: This sets the handler for resolving the CSRF token from the request. `CsrfTokenRequestAttributeHandler` is a common choice.
*   **`ignoringRequestMatchers(...)`**: This is the crucial part for Vaadin integration. By using `ignoringRequestMatchers`, you tell Spring Security to bypass CSRF checks for requests matching the specified Ant-style paths. These paths (`/VAADIN/**`, `/VAADIN/push/**`, `/hilla/**`, etc.) are used by Vaadin for its internal communication, and Vaadin handles CSRF for these requests internally. If you don't ignore these, Spring Security would block Vaadin's requests, leading to a broken application.
*   **`authorizeHttpRequests(...)`**: This section defines your authorization rules. You need to explicitly permit access to Vaadin's internal resources (`/VAADIN/**`, `/hilla/**`) and any other public resources (like your login page, static assets) that should be accessible without authentication.
*   **`formLogin(...)` and `logout(...)`**: These configure your login and logout mechanisms, similar to a standard Spring Security setup.

**Important Considerations for Manual Configuration:**

*   **Complexity:** Manually configuring CSRF for Vaadin can be more complex and error-prone than extending `VaadinWebSecurity`. It requires a deeper understanding of both Spring Security's and Vaadin's internal workings.
*   **Maintenance:** Future Vaadin or Spring Security updates might introduce changes to internal paths or CSRF handling, requiring you to update your manual configuration. `VaadinWebSecurity` is designed to abstract away these changes.
*   **Hilla Specifics:** For Hilla applications, ensure you ignore paths relevant to Hilla endpoints and development mode (`/hilla/**`, `/hilla-dev-mode-gizmo/**`).
*   **Custom Endpoints:** If you have custom REST controllers in your Spring Boot application that are *not* part of Vaadin's UI communication, Spring Security's CSRF protection will still apply to them (unless explicitly ignored). You would then need to handle CSRF tokens for those endpoints on your client-side (e.g., using React as discussed previously).

In summary, while it's possible to manually configure Spring Security for CSRF with Vaadin without extending `VaadinWebSecurity`, it's generally not recommended due to increased complexity and maintenance overhead. `VaadinWebSecurity` is provided specifically to simplify this integration and ensure compatibility.

