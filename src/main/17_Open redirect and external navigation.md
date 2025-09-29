# 17_Open redirect and external navigation

> Problem vs Solved code patterns with rollout checks and attacker playbooks.
> 

### Goal

Harden open redirects and external navigation to prevent phishing pivots, OAuth token leakage, and reverse tabnabbing.

---

### 1) Dynamic redirects → tokenized allow‑list destinations

### How attackers exploit this

- Manipulate a next or redirect param to send users to a phishing site after login.
- Abuse an open redirector with OAuth or OIDC to pivot authorization codes or tokens.
- Chain open redirect with XSS to exfiltrate credentials.

### Before: naive "next" parameter (open redirect)

```java
// Problem: trusts user-controlled URL
@GetMapping("/login/success")
public String afterLogin(@RequestParam(required = false) String next) {
    String target = (next != null) ? next : "/";
    return "redirect:" + target;  // Open redirect! e.g., ?next=[https://evil.example](https://evil.example)
}
```

### After A: map short tokens to an allow‑list (server-side only)

```java
// A simple allow-list map (env- or config-backed in real code)
private static final Map<String, String> DESTS = Map.of(
    "home", "/",
    "profile", "/account",
    "docs", "[https://docs.example.com](https://docs.example.com)"
);

@GetMapping("/login/success")
public String afterLogin(@RequestParam(required = false) String dest) {
    String safe = DESTS.getOrDefault(dest, "/");
    return "redirect:" + safe;
}
```

### After B: signed destination tokens (resistant to tampering)

- Issue a short-lived, HMAC-signed token that encodes a destination key.
- Verify signature and expiry before redirect.

```java
// Token payload: { "d":"profile", "exp":1735689600 } and HMAC-SHA256 signature
record DestToken(String d, long exp, String sig) {}

boolean verify(DestToken t) {
    if ([Instant.now](http://Instant.now)().getEpochSecond() > t.exp) return false;
    String msg = t.d + "." + t.exp;
    String expected = hmacSha256Base64(secret, msg);
    return MessageDigest.isEqual(expected.getBytes(StandardCharsets.UTF_8),
                                 t.sig.getBytes(StandardCharsets.UTF_8));
}

@GetMapping("/login/success")
public String afterLogin(@RequestParam(required = false) String t) {
    DestToken tok = parseToken(t); // parse from URL-safe form
    if (tok != null && verify(tok) && DESTS.containsKey(tok.d())) {
        return "redirect:" + DESTS.get(tok.d());
    }
    return "redirect:/"; // default on failure
}
```

Key points

- Never accept arbitrary absolute URLs.
- Prefer server-known keys to full URLs.
- If a full external URL must be supported, pin exact hosts and enforce https with strict equality, not startsWith.

---

### 2) target="_blank" without rel → reverse tabnabbing

### How attackers exploit this

- The opened page can call `window.opener.location = '[https://phish.example](https://phish.example)'` to replace your original tab with a phishing clone.

### Before: missing rel attributes

```html
<!-- Vulnerable to reverse tabnabbing -->
<a href="[https://docs.example.com](https://docs.example.com)" target="_blank">Docs</a>
```

### After: add rel="noopener noreferrer"

```html
<a href="[https://docs.example.com](https://docs.example.com)" target="_blank" rel="noopener noreferrer">Docs</a>
```

Reusable React helper

```tsx
type ExtLinkProps = React.AnchorHTMLAttributes<HTMLAnchorElement>;
export function ExtLink({ children, ...props }: ExtLinkProps) {
  return (
    <a target="_blank" rel="noopener noreferrer" {...props}>
      {children}
    </a>
  );
}
```

Server-side template example (Thymeleaf)

```html
<a th:href="@{${url}}" target="_blank" rel="noopener noreferrer" th:text="${label}"></a>
```

---

### 3) Intermediate warning page for allowed external redirects

Why

- Prevents surprise context switches and reduces phishing risk.
- Gives users a chance to back out.
- Enables audit logging.

Flow

1. Backend validates tokenized destination. If external, render a "Leaving site" page.
2. Page shows the exact destination host, reason, and a continue button.

Controller

```java
@GetMapping("/go")
public String go(@RequestParam String dest, Model model) {
    String url = DESTS.get(dest);
    if (url == null) return "redirect:/";
    URI uri = URI.create(url);

    boolean external = uri.isAbsolute() && !uri.getHost().endsWith("[example.com](http://example.com)");
    if (external) {
        model.addAttribute("host", uri.getHost());
        model.addAttribute("url", url);
        // Optionally log userId, dest, timestamp
        return "leaving"; // renders leaving.html
    }
    return "redirect:" + url;
}
```

Template (leaving.html)

```html
<!doctype html>
<html lang="en">
<head>
  <meta http-equiv="Content-Security-Policy" content="default-src 'self'">
  <title>Leaving Example</title>
</head>
<body>
  <h1>You are leaving [example.com](http://example.com)</h1>
  <p>You are about to visit: <code id="destHost">[[${host}]]</code></p>
  <p>Only proceed if you trust this site.</p>
  <p><a id="continue" href="[[${url}]]" rel="noopener noreferrer">Continue</a></p>
  <p><a href="/">Go back</a></p>
</body>
</html>
```

Hardening

- Show the hostname prominently. Avoid auto-redirect; require user click.
- Add rel="noopener noreferrer" on the continue link.
- CSP default-src 'self' to keep the page inert.
- Log the event for monitoring.

---

### 4) Rigid validation when you must accept a URL

```java
private static final Set<String> ALLOWED_HOSTS = Set.of(
    "[docs.example.com](http://docs.example.com)",
    "[status.example.com](http://status.example.com)",
    "[support.vendorA.com](http://support.vendorA.com)"
);

boolean isAllowedExternal(URI uri) {
    if (!"https".equalsIgnoreCase(uri.getScheme())) return false;
    String host = uri.getHost();
    if (host == null) return false;

    // Exact host match only. Avoid wildcard pitfalls.
    return ALLOWED_HOSTS.contains(host);
}

@GetMapping("/redir")
public String redir(@RequestParam String url) {
    URI uri = URI.create(url);
    if (isAllowedExternal(uri)) {
        return "redirect:" + uri.toString();
    }
    return "redirect:/";
}
```

Pitfalls to avoid

- startsWith or contains checks on strings.
- Allowing `//evil.example` or protocol-relative URLs.
- Not re-parsing after decoding. Normalize and validate once.
- Not requiring https.

---

### 5) Tests and quick checks

Unit tests

```java
@Test
void rejects_external_non_allowlisted() {
    var res = ctrl.redir("[https://evil.example](https://evil.example)");
    assertEquals("redirect:/", res);
}

@Test
void accepts_allowlisted() {
    var res = ctrl.redir("[https://docs.example.com](https://docs.example.com)");
    assertTrue(res.startsWith("redirect:[https://docs.example.com](https://docs.example.com)"));
}

@Test
void blocks_protocol_relative() {
    var res = ctrl.redir("//evil.example");
    assertEquals("redirect:/", res);
}
```

E2E checks

- Try ../ and double-encoding tricks like %2e%2e, %2F, mixed case schemes, and Unicode dots.
- Validate that login success cannot be coerced to leave the site via next or dest.

---

### 6) Rollout checklist

- Replace any redirect that uses user-provided absolute URLs with token-to-URL mapping.
- Add rel="noopener noreferrer" to every target="_blank".
- Route external destinations through a Leaving page with logging.
- Add unit and integration tests for redirect validation.
- Monitor logs for attempted external destinations and suspicious hosts.