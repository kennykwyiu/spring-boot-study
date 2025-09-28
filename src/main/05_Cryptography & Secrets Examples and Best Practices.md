# Cryptography & Secrets: Examples and Best Practices

> Practical examples of insecure code patterns and secure alternatives for cryptography and secret management.
> 

### Use of Non-Cryptographic Random

- Problem
    
    Using java.util.Random for security-related values like session IDs, tokens, password reset codes, or crypto keys is predictable.
    

```java
// Insecure: predictable session ID
import java.util.Random;

Random random = new Random();
long sessNum = random.nextLong();
String sessionId = Long.toHexString(sessNum);
```

- Best practice
    
    Use a CSPRNG such as [java.security](http://java.security).SecureRandom. Prefer byte arrays with sufficient length. Avoid converting raw bytes to String without encoding.
    

```java
// Secure: 256-bit random token
import [java.security](http://java.security).SecureRandom;
import java.util.Base64;

SecureRandom sr = new SecureRandom();
byte[] token = new byte[32]; // 256 bits
sr.nextBytes(token);
String sessionId = Base64.getUrlEncoder().withoutPadding().encodeToString(token);
```

- Additional tips
    - Do not seed SecureRandom manually unless you provide high-entropy seed material.
    - Ensure tokens are long enough to prevent brute force (>= 128 bits; 256 bits preferred for long-lived tokens).

---

### Spring Use of Hardcoded Password

- Problem
    
    Embedding passwords or secrets directly in code or annotations leaks credentials and is costly to rotate.
    

```java
// Insecure: hardcoded secret in code
@Value("Password123!")
private String adminPassword;

// Insecure: default fallback that leaves a secret in code
@Value("${ADMIN_PASSWORD:Benfica123}")
private String defaultedPassword;

// Insecure: in-memory auth with plaintext
auth.inMemoryAuthentication()
    .withUser("admin")
    .password("{noop}password")
    .roles("USER", "ADMIN");
```

- Best practice
    
    Use externalized configuration with environment variables or a secrets manager. Use password encoders and never store plaintext.
    

```java
// application.yml (do not commit real secrets; use env or vault)
security:
  admin:
    user: ${ADMIN_USER}
    password: ${ADMIN_PASSWORD}

// Java config: inject from env/secret store
@Value("${ADMIN_USER}")
private String adminUser;
@Value("${ADMIN_PASSWORD}")
private String adminPassword;

@Bean
[org.springframework.security](http://org.springframework.security).crypto.password.PasswordEncoder passwordEncoder() {
  return new [org.springframework.security](http://org.springframework.security).crypto.bcrypt.BCryptPasswordEncoder();
}

// Example: storing only hashed passwords (e.g., in DB)
String hashed = passwordEncoder().encode(rawPassword);
// Compare on login via AuthenticationManager rather than manual string compare
```

- Recommended approaches
    - Use a secrets manager: AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault, Azure Key Vault.
    - Disable committing secrets: pre-commit hooks, repo scanners, and CI checks.
    - Rotate credentials regularly and enforce least privilege.

---

### TruffleHog High-Entropy Strings (Secret Leakage)

- Problem
    
    High-entropy strings in repositories often indicate leaked credentials, API keys, or tokens.
    
- Example of an accidental leak

```java
// Insecure: leaked API key committed to repo
private static final String STRIPE_API_KEY = "sk_live_51KcS...RANDOMHIGHENTROPY...";
```

- Best practice: detection and response
    
    1) Scan
    
    - Use TruffleHog, GitLeaks, or git-secrets in CI to block commits containing secrets.
    
    2) If a secret is found
    
    - Revoke or rotate the secret immediately in the issuing system.
    - Purge from history if required: git filter-repo or git filter-branch.
    - Add detection rules and commit hooks to prevent recurrence.
- Example CI snippet (GitHub Actions)

```yaml
name: secret-scan
on: [push, pull_request]
jobs:
  trufflehog:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run TruffleHog
        uses: trufflesecurity/trufflehog@main
        with:
          path: .
          extra_args: --only-verified
```

---

### General Hardening Checklist

- Centralize secret management and never hardcode secrets in code or Docker images.
- Limit secret exposure to the minimum scope and lifetime required.
- Use role-based access and short-lived credentials where possible.
- Log secret access events, not secret values. Mask in logs and UIs.
- Ensure backups and crash dumps do not contain plaintext secrets.
- Review PRs for secret handling patterns and add automated checks.