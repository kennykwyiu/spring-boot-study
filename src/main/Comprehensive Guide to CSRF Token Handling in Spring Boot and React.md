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

