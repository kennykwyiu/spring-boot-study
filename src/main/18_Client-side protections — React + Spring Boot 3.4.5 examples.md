# Client-side protections — React + Spring Boot 3.4.5 examples

### Client-side protections — React + Spring Boot 3.4.5 examples

This page provides “problem” vs “solved” code demonstrating:

- CSP with per-request nonce on scripts
- Strict allow-lists for script, image, and connect
- frame-ancestors 'none' by default (or allow-list specific embeds)
- Optional legacy frame-buster for very old browsers only

---

### Project layout

- Backend: Spring Boot 3.4.5 (serves HTML shell and sets headers)
- Frontend: React build served as static assets from Spring Boot
- Template engine: Thymeleaf to inject the nonce into script tags

---

### Problem version (vulnerable)

Symptoms: No CSP. Inline scripts without nonce. No frame protection. Unrestricted fetch.

src/main/resources/templates/index_bad.html

```html
<!doctype html>
<html lang="en">
<head>
	<meta charset="utf-8" />
	<title>React App (Bad)</title>
</head>
<body>
	<div id="root"></div>
	<script>console.log('inline without CSP');</script>
	<script src="[https://untrusted.example/any.js"></script>](https://untrusted.example/any.js"></script>)
	<script type="module" src="/static/assets/index.js"></script>
</body>
</html>
```

src/main/java/com/example/web/[IndexController.java](http://IndexController.java)

```java
package com.example.web;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class IndexController {
	@GetMapping({"/", "/app/**"})
	public String index() {
		return "index_bad"; // no headers, no nonce
	}
}
```

No security filter or headers. Page can be framed by any site. Scripts can execute from anywhere.

---

### Solved version (secure)

Key ideas: Generate a nonce per request. Set CSP including that nonce and tight allow-lists. Deny framing. Add key security headers. Attach nonce to every script.

build.gradle.kts

```kotlin
plugins {
    id("java")
    id("org.springframework.boot") version "3.4.5"
    id("io.spring.dependency-management") version "1.1.6"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-thymeleaf")
    implementation("org.springframework.boot:spring-boot-starter-security")
}

java { toolchain { languageVersion.set(JavaLanguageVersion.of(21)) } }
```

src/main/java/com/example/security/[NonceUtil.java](http://NonceUtil.java)

```java
package [com.example.security](http://com.example.security);

import [java.security](http://java.security).SecureRandom;
import java.util.Base64;

public final class NonceUtil {
	private static final SecureRandom RNG = new SecureRandom();
	public static String newNonce() {
		byte[] b = new byte[16];
		RNG.nextBytes(b);
		return Base64.getEncoder().encodeToString(b);
	}
	private NonceUtil() {}
}
```

src/main/java/com/example/security/[CspNonceFilter.java](http://CspNonceFilter.java)

```java
package [com.example.security](http://com.example.security);

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;
import [java.io](http://java.io).IOException;

@Component
@Order(0)
public class CspNonceFilter extends OncePerRequestFilter {
	@Override
	protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
			throws ServletException, IOException {
		String nonce = NonceUtil.newNonce();
		req.setAttribute("cspNonce", nonce);

		String self = "'self'";
		String cdn = "[https://cdn.example.com](https://cdn.example.com)"; // adjust
		String api = "[https://api.example.com](https://api.example.com)"; // adjust

		String csp = String.join("; ",
			"default-src 'none'",
			"base-uri " + self,
			"frame-ancestors 'none'", // or allow-list exact origins
			"script-src " + self + " " + cdn + " 'nonce-" + nonce + "'",
			"script-src-attr 'none'",
			"script-src-elem " + self + " " + cdn + " 'nonce-" + nonce + "'",
			"style-src " + self,
			"img-src " + self + " data:",
			"font-src " + self,
			"connect-src " + self + " " + api,
			"object-src 'none'",
			"frame-src 'none'",
			"form-action " + self,
			"worker-src " + self,
			"upgrade-insecure-requests"
		);

		res.setHeader("Content-Security-Policy", csp);
		res.setHeader("X-Frame-Options", "DENY");
		res.setHeader("Referrer-Policy", "no-referrer");
		res.setHeader("Permissions-Policy", "geolocation=(), camera=(), microphone=()");
		res.setHeader("X-Content-Type-Options", "nosniff");
		res.setHeader("Strict-Transport-Security", "max-age=31536000; includeSubDomains; preload");
		res.setHeader("Cross-Origin-Opener-Policy", "same-origin");
		res.setHeader("Cross-Origin-Resource-Policy", "same-origin");

		chain.doFilter(req, res);
	}
}
```

src/main/java/com/example/security/[SecurityConfig.java](http://SecurityConfig.java)

```java
package [com.example.security](http://com.example.security);

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import [org.springframework.security.config.annotation.web.builders](http://org.springframework.security.config.annotation.web.builders).HttpSecurity;
import [org.springframework.security](http://org.springframework.security).web.SecurityFilterChain;

@Configuration
public class SecurityConfig {
	@Bean
	SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
			.csrf(csrf -> csrf.disable()) // enable & align if using cookie auth
			.authorizeHttpRequests(auth -> auth
				.requestMatchers("/static/**", "/assets/**").permitAll()
				.anyRequest().permitAll()
			)
			.headers(h -> h.frameOptions(f -> f.deny()));
		return [http.build](http://http.build)();
	}
}
```

src/main/java/com/example/web/[IndexController.java](http://IndexController.java)

```java
package com.example.web;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class IndexController {
	@GetMapping({"/", "/app/**"})
	public String index(HttpServletRequest req, Model model) {
		Object nonce = req.getAttribute("cspNonce");
		model.addAttribute("nonce", nonce != null ? nonce.toString() : "");
		return "index_secure";
	}
}
```

src/main/resources/templates/index_secure.html

```html
<!doctype html>
<html lang="en" xmlns:th="[http://www.thymeleaf.org](http://www.thymeleaf.org)">
<head>
	<meta charset="utf-8" />
	<meta name="viewport" content="width=device-width, initial-scale=1" />
	<title>React App (Secure)</title>
</head>
<body>
	<div id="root"></div>

	<!-- Optional legacy frame-buster: only if supporting very old browsers -->
	<script th:attr="nonce=${nonce}">
		if (self !== top) { try { top.location = self.location; } catch(e) {} }
	</script>

	<!-- React bundle emitted to classpath:/static/assets/index.js -->
	<script type="module" th:attr="nonce=${nonce}" src="/static/assets/index.js"></script>
</body>
</html>
```

src/main/resources/application.yml

```yaml
spring:
  web:
    resources:
      static-locations: classpath:/static/
```

Frontend bootstrap (bundled to /static/assets/index.js)

```jsx
// src/main/frontend/App.jsx
import React, { useEffect } from 'react';

export default function App() {
	useEffect(() => {
		// Allowed by connect-src in CSP
		fetch('[https://api.example.com/ping](https://api.example.com/ping)', { credentials: 'omit' })
			.then(r => console.log('API status', r.status))
			.catch(e => console.error('API error', e));
	}, []);
	return <main><h1>Secure React + Spring Boot</h1></main>;
}
```

```jsx
// src/main/frontend/main.jsx
import React from 'react';
import { createRoot } from 'react-dom/client';
import App from './App.jsx';

createRoot(document.getElementById('root')).render(<App />);
```

Vite config to emit deterministic name

```jsx
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
	plugins: [react()],
	build: {
		outDir: 'build-static',
		assetsDir: 'assets',
		rollupOptions: { output: { entryFileNames: 'index.js' } }
	}
});
```

Gradle copy into classpath static

```kotlin
// build.gradle.kts
import org.gradle.api.tasks.Copy

tasks.register<Copy>("copyFrontend") {
	from("build-static")
	into("src/main/resources/static")
}

tasks.named("processResources") { dependsOn("copyFrontend") }
```

---

### Validation checklist

- Scripts execute only when carrying the server-issued nonce or hosted on allow-listed origins
- No third-party images load unless listed in img-src
- fetch calls fail unless target matches connect-src
- Page cannot be framed by other origins due to frame-ancestors 'none' and X-Frame-Options DENY