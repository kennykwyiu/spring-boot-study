# Input Validation — Before & After Examples

> Practical, copy-pastable examples for common input-validation flaws. Each section shows unsafe "Before" code and a safer "After" version, plus key practices.
> 

### 1) Input Path Not Canonicalized

- Risk: User-controlled file paths can reference unexpected files if not normalized and constrained.

**Before — Java (absolute path use, no canonicalization)**

```java
private String readFile(HttpServletRequest req) throws IOException {
	String filename = req.getParameter("filename");
	Path path = Paths.get(filename);                 // may be absolute or outside allowed dir
	return Files.readString(path);
}
```

**After — Canonicalize + restrict to an allowlisted base directory**

```java
private static final Path BASE = Paths.get("/app/data").toAbsolutePath().normalize();

private static String sanitizeFilename(String input) {
	return Paths.get(input).getFileName().toString(); // drop any path components
}

private String readFileSafe(HttpServletRequest req) throws IOException {
	String raw = req.getParameter("filename");
	String safeName = sanitizeFilename(raw);
	Path target = BASE.resolve(safeName).normalize();
	if (!target.startsWith(BASE)) {
		throw new SecurityException("Invalid path");
	}
	return Files.readString(target);
}
```

Key practices:

- Treat only the file name as dynamic, not directories
- Canonicalize and compare against an allowed base folder
- Use least-privilege filesystem permissions

---

### 2) Relative Path Traversal

- Risk: `../` sequences can escape intended directories and access sensitive files.

**Before — Java (relative join vulnerable to traversal)**

```java
private String readFile(HttpServletRequest req) throws IOException {
	String filename = req.getParameter("filename");
	Path path = Paths.get("static/" + filename); // "../../etc/passwd"
	return Files.readString(path);
}
```

**After — Strip traversal + enforce base folder**

```java
private static final Path PUBLIC_DIR = Paths.get("static").toAbsolutePath().normalize();

private static String basename(String input) {
	return Paths.get(input).getFileName().toString();
}

private String readFileSafe(HttpServletRequest req) throws IOException {
	String fileParam = basename(req.getParameter("filename"));
	Path target = PUBLIC_DIR.resolve(fileParam).normalize();
	if (!target.startsWith(PUBLIC_DIR)) throw new SecurityException("Traversal detected");
	return Files.readString(target);
}
```

Key practices:

- Normalize and compare canonical paths
- Reject any path that escapes the allowed directory
- Maintain an allowlist if files are finite and known

---

### 3) Cleansing Canonicalization and Comparison Errors

- Risk: Security checks on raw input or non-canonical forms can be bypassed via encodings or path tricks.

**Before — Block by naive prefix check**

```java
boolean isForbidden(HttpServletRequest req) {
	String uri = req.getRequestURI();
	return uri.startsWith("/admin");           // "/NOTEXISTS/../admin" bypasses
}
```

**After — Canonicalize, then check policy**

```java
import org.owasp.encoder.Encode; // or an equivalent canonicalizer

boolean isForbidden(HttpServletRequest req) {
	String raw = req.getRequestURI();
	String canon = ESAPI.encoder().canonicalize(raw); // or a trusted canonicalizer
	String normalized = Paths.get(canon).normalize().toString();
	return normalized.equals("/admin") || normalized.startsWith("/admin/");
}
```

Alternative hardening:

- Use router or framework route guards instead of string checks
- Avoid making authz decisions from user-controlled strings; prefer server-side identity and roles

---

### 4) Unrestricted File Upload

- Risk: Large or malicious uploads can exhaust storage or lead to RCE if executed.

Threats to control:

- Size: enforce max request and per-file limits
- Type: validate content-type by magic numbers, not just extensions
- Storage: store outside web-root, randomize names, never execute

**Before — Accept and store without checks**

```java
@PostMapping("/upload")
public String upload(@RequestParam("file") MultipartFile file) throws IOException {
	Path dest = Paths.get("/var/www/uploads/" + file.getOriginalFilename());
	Files.copy(file.getInputStream(), dest);
	return "ok";
}
```

**After — Size limit, type allowlist, safe storage**

```java
private static final long MAX_BYTES = 5L * 1024 * 1024; // 5 MB
private static final Set<String> ALLOWED = Set.of("image/png","image/jpeg","application/pdf");
private static final Path STORAGE = Paths.get("/srv/app/uploads").toAbsolutePath().normalize();

@PostMapping("/upload")
public String upload(@RequestParam("file") MultipartFile file) throws IOException {
	if (file.isEmpty() || file.getSize() > MAX_BYTES) {
		throw new IllegalArgumentException("Invalid size");
	}
	String contentType = Optional.ofNullable(file.getContentType()).orElse("");
	if (!ALLOWED.contains(contentType)) {
		throw new IllegalArgumentException("Invalid type");
	}
	String safeName = UUID.randomUUID() + safeExt(contentType); // do not trust original name
	Path dest = STORAGE.resolve(safeName).normalize();
	Files.createDirectories(STORAGE);
	try (InputStream in = file.getInputStream()) {
		Files.copy(in, dest, StandardCopyOption.REPLACE_EXISTING);
	}
	return "ok";
}

private static String safeExt(String ct) {
	return switch (ct) {
		case "image/png" -> ".png";
		case "image/jpeg" -> ".jpg";
		case "application/pdf" -> ".pdf";
		default -> "";
	};
}
```

Deployment tips:

- Set container limits: max request size, max file size, streaming upload
- Scan uploads out-of-band; quarantine on fail
- Never place uploads under a path that the server executes or serves without additional controls

---

### Quick Checklist

- Normalize and compare canonical forms for paths and URLs
- Keep user input out of authorization decisions where possible
- Constrain file uploads: size, type, and storage location
- Apply least privilege on the filesystem and the app user

Related references on this workspace:

- See category 3 “Input Validation” under Risk Cat categorization for context [Risk Cat categorization](https://www.notion.so/Risk-Cat-categorization-1e14532e839b8007bcf6e2cc33ea7e85?pvs=21)