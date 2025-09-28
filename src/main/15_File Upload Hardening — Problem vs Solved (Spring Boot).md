# File Upload Hardening — Problem vs Solved (Spring Boot)

### Overview

Hardening file uploads in Spring Boot 3.x with copy‑ready bad vs. good examples, tests for traversal, polyglot, oversize, and optional AV integration.

---

### 1) Limit size and concurrent uploads

How attackers exploit

- Upload DoS: very large files exhaust memory or disk
- Slowloris body: many concurrent slow uploads tie up threads and I/O
- Zip-bombs or nested archives explode server resources

Problem code

```java
@PostMapping("/upload")
public String upload(@RequestParam("file") MultipartFile file) throws IOException {
	// No size checks at framework, servlet, or application layer
	Path dest = Path.of("/var/www/uploads/" + file.getOriginalFilename());
	Files.copy(file.getInputStream(), dest);
	return "ok";
}
```

Solved code

- Enforce limits at:
    - Container: server.tomcat.max-swallow-size, max-http-form-post-size
    - Spring: spring.servlet.multipart.max-file-size, max-request-size
    - App layer: explicit checks
    - Concurrency: executor limits and backpressure

application.yml

```yaml
server:
  tomcat:
    max-swallow-size: 10MB
    max-http-form-post-size: 12MB
spring:
  servlet:
    multipart:
      max-file-size: 8MB
      max-request-size: 8MB
```

Controller

```java
@RestController
class UploadController {
	private static final long MAX_BYTES = 8L * 1024 * 1024; // 8MB

	@PostMapping("/upload")
	public ResponseEntity<?> upload(@RequestParam("file") MultipartFile file) throws IOException {
		if (file.isEmpty() || file.getSize() > MAX_BYTES) {
			return ResponseEntity.badRequest().body("Invalid size");
		}
		// Stream with NIO and fail if exceeds threshold (defense in depth)
		try (InputStream in = new BoundedInputStream(file.getInputStream(), MAX_BYTES)) {
			// ... pass stream to safe storage (see below)
		}
		return ResponseEntity.ok("ok");
	}
}
```

Concurrency control (example bean)

```java
@Configuration
class ExecConfig {
	@Bean
	public TaskExecutor uploadExecutor() {
		ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
		ex.setCorePoolSize(8);
		ex.setMaxPoolSize(16);
		ex.setQueueCapacity(100);
		ex.setThreadNamePrefix("upload-");
		ex.initialize();
		return ex;
	}
}
```

---

### 2) Validate extension AND magic bytes

How attackers exploit

- Polyglots: .jpg that is also PHP or JS payload
- Mismatch: rename .exe to .png to bypass naive extension filters

Problem code

```java
String name = file.getOriginalFilename();
if (!name.endsWith(".png") && !name.endsWith(".jpg")) {
	return "bad type";
}
// Trusting client-provided MIME and extension only
```

Solved code

- Check both: allowlist extension and server-side magic bytes sniffing
- Optionally re-encode images to known-safe format

Sniff utility

```java
final class Magic {
	static boolean looksLikePng(InputStream in) throws IOException {
		in.mark(8);
		byte[] sig = in.readNBytes(8);
		in.reset();
		byte[] png = new byte[]{(byte)137,80,78,71,13,10,26,10};
		return Arrays.equals(sig, png);
	}
	static boolean looksLikeJpeg(InputStream in) throws IOException {
		in.mark(2);
		byte[] sig = in.readNBytes(2);
		in.reset();
		return sig.length == 2 && sig[0] == (byte)0xFF && sig[1] == (byte)0xD8;
	}
}
```

Controller snippet

```java
String original = Optional.ofNullable(file.getOriginalFilename()).orElse("");
String ext = original.contains(".") ? original.substring(original.lastIndexOf('.') + 1).toLowerCase() : "";
Set<String> allowed = Set.of("png","jpg","jpeg");

if (!allowed.contains(ext)) {
	return ResponseEntity.badRequest().body("Extension not allowed");
}
try (InputStream in = file.getInputStream()) {
	boolean ok = switch (ext) {
		case "png" -> Magic.looksLikePng(in);
		case "jpg", "jpeg" -> Magic.looksLikeJpeg(in);
		default -> false;
	};
	if (!ok) return ResponseEntity.badRequest().body("Magic bytes mismatch");
}
```

Optional re-encode images

```java
BufferedImage img = [ImageIO.read](http://ImageIO.read)(file.getInputStream());
if (img == null) return ResponseEntity.badRequest().body("Not an image");
try (OutputStream out = Files.newOutputStream(tmpPath)) {
	ImageIO.write(img, "png", out); // normalize to PNG
}
```

---

### 3) Store outside webroot with random names

How attackers exploit

- Upload a web shell to a path executed by the server
- Predictable name paths allow direct retrieval or overwrite attacks

Problem code

```java
Path dest = Path.of("/var/www/html/uploads/" + file.getOriginalFilename()); // in webroot, predictable
Files.copy(file.getInputStream(), dest, StandardCopyOption.REPLACE_EXISTING);
return "/uploads/" + file.getOriginalFilename();
```

Solved code

- Store outside webroot
- Random, unguessable filename
- Keep original filename as metadata only
- Serve files via controller with Content-Disposition, or signed URLs from object storage

Storage

```java
Path SAFE_ROOT = Path.of("/srv/appdata/uploads"); // not served by web server
Files.createDirectories(SAFE_ROOT);

String safeExt = ext; // computed earlier and allowlisted
String randomName = UUID.randomUUID().toString().replace("-", "") + "." + safeExt;
Path dest = SAFE_ROOT.resolve(randomName);

// Stream copy
try (InputStream in = file.getInputStream();
     OutputStream out = Files.newOutputStream(dest, StandardOpenOption.CREATE_NEW)) {
	in.transferTo(out);
}

// Persist metadata in DB: id, randomName, originalName, size, sha256, contentType
```

Serving download

```java
@GetMapping("/files/{id}")
public ResponseEntity<Resource> get(@PathVariable String id) throws IOException {
	// lookup randomName by id + authorization checks
	Path path = SAFE_ROOT.resolve(randomName);
	ByteArrayResource res = new ByteArrayResource(Files.readAllBytes(path));
	return ResponseEntity.ok()
		.header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + originalName.replace("\"","") + "\"")
		.contentType(MediaType.APPLICATION_OCTET_STREAM)
		.body(res);
}
```

---

### 4) Canonicalize paths, never trust client paths

How attackers exploit

- Path traversal: filename like ../../../../etc/passwd or ..\..\webapps\ROOT\shell.jsp
- Unicode tricks: mixed separators, percent-encoding to escape allowed directories

Problem code

```java
Path base = Path.of("/srv/appdata/uploads");
Path dest = base.resolve(file.getOriginalFilename()); // directly using client path
Files.copy(file.getInputStream(), dest);
```

Solved code

- Ignore client path components completely
- If you must use subfolders, canonicalize and enforce base containment

Safe resolve

```java
Path base = Path.of("/srv/appdata/uploads");
Files.createDirectories(base);

String randomName = UUID.randomUUID().toString().replace("-", "") + "." + safeExt;
Path dest = base.resolve(randomName).normalize();

if (!dest.startsWith(base)) {
	throw new SecurityException("Path breakout");
}
try (InputStream in = file.getInputStream();
     OutputStream out = Files.newOutputStream(dest, StandardOpenOption.CREATE_NEW)) {
	in.transferTo(out);
}
```

---

### 5) AV scan or sandbox for risky types

How attackers exploit

- Malware distribution, credential stealers
- Exploit readers of office/PDF/media formats
- Server-side image parsers with known CVEs

Problem code

```java
// Accept any file and make it immediately available to users with no scanning
```

Solved code

- Use on-access AV like ClamAV daemon or vendor engine
- Quarantine until clean
- For archives: scan both container and extracted contents with bounded decompression

ClamAV example (simplified)

```java
@Component
class ClamAvScanner {
	private final String host = "[localhost](http://localhost)";
	private final int port = 3310;

	boolean scan(Path file) throws IOException {
		try (Socket sock = new Socket(host, port);
		     OutputStream out = sock.getOutputStream();
		     InputStream in = sock.getInputStream()) {
			out.write("zINSTREAM\0".getBytes([StandardCharsets.US](http://StandardCharsets.US)_ASCII));
			try (InputStream fin = Files.newInputStream(file)) {
				byte[] buf = new byte[8192];
				int r;
				while ((r = [fin.read](http://fin.read)(buf)) != -1) {
					out.write(ByteBuffer.allocate(4).putInt(r).array());
					out.write(buf, 0, r);
				}
			}
			out.write(new byte[]{0,0,0,0}); // end
			out.flush();
			String resp = new String(in.readNBytes(512), [StandardCharsets.US](http://StandardCharsets.US)_ASCII);
			return resp.contains("OK");
		}
	}
}
```

Usage

```java
Path dest = /* saved file */;
boolean clean = clamAvScanner.scan(dest);
if (!clean) {
	Files.deleteIfExists(dest);
	return ResponseEntity.status(422).body("Failed AV scan");
}
```

---

### Extra hardening checklist

- Compute and store SHA-256 of the content. Use it to dedupe and to verify integrity on serve
- Strip metadata from images and PDFs if privacy is a concern
- Treat content-type from client as untrusted. Set your own when serving
- For images, consider transcoding to a safe, known set and bounding dimensions to prevent decompression bomb images
- Log request ID, user ID, size, type, hash, and outcome. Avoid logging raw filenames or paths from the client without sanitization
- Apply authorization per file ID. Do not serve by raw filename
- For archives, use a safe extractor that:
    - Enforces per-entry size and total size ceilings
    - Rejects absolute paths and traversal
    - Limits nesting depth and entry count

---

### Hardened upload controller (finalized)

Minimal build config

```xml
<!-- pom.xml -->
<dependency>
	<groupId>commons-io</groupId>
	<artifactId>commons-io</artifactId>
	<version>2.15.1</version>
</dependency>
```

Controller

```java
// src/main/java/com/example/upload/[SafeUploadController.java](http://SafeUploadController.java)
package com.example.upload;

import [org.apache.commons.io](http://org.apache.commons.io).input.BoundedInputStream;
import [org.springframework.core.io](http://org.springframework.core.io).ByteArrayResource;
import [org.springframework.core.io](http://org.springframework.core.io).Resource;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import [java.io](http://java.io).*;
import [java.net](http://java.net).Socket;
import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;
import java.util.*;

@RestController
@RequestMapping("/upload")
public class SafeUploadController {
	private static final long MAX_BYTES = 8L * 1024 * 1024; // 8 MB
	private static final Set<String> ALLOWED = Set.of("png","jpg","jpeg","pdf");
	private static final Path SAFE_ROOT = Path.of(System.getProperty("upload.dir", "/srv/appdata/uploads"));

	private final Optional<ClamAvScanner> av;

	public SafeUploadController(Optional<ClamAvScanner> av) {
		this.av = av;
	}

	@PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
	public ResponseEntity<?> upload(@RequestParam("file") MultipartFile file) throws IOException {
		if (file.isEmpty() || file.getSize() > MAX_BYTES) {
			return ResponseEntity.badRequest().body(Map.of("error","Invalid size"));
		}

		String original = Optional.ofNullable(file.getOriginalFilename()).orElse("file");
		String ext = extractExt(original);
		if (!ALLOWED.contains(ext)) return ResponseEntity.badRequest().body(Map.of("error","Extension not allowed"));

		Files.createDirectories(SAFE_ROOT);
		String randomName = UUID.randomUUID().toString().replace("-", "") + "." + ext;
		Path dest = SAFE_ROOT.resolve(randomName).normalize();
		if (!dest.startsWith(SAFE_ROOT)) throw new SecurityException("Path breakout");

		// Magic bytes checks
		try (InputStream sniff = file.getInputStream()) {
			if (!magicOk(sniff, ext)) return ResponseEntity.badRequest().body(Map.of("error","Magic bytes mismatch"));
		}

		// Save bounded
		try (InputStream in = new BoundedInputStream(file.getInputStream(), MAX_BYTES);
		     OutputStream out = Files.newOutputStream(dest, StandardOpenOption.CREATE_NEW)) {
			in.transferTo(out);
		}

		// Optional AV
		if (av.isPresent() && !av.get().scan(dest)) {
			Files.deleteIfExists(dest);
			return ResponseEntity.status(422).body(Map.of("error","Malicious content detected"));
		}

		String id = randomName.substring(0, randomName.indexOf('.'));
		return ResponseEntity.ok(Map.of("id", id, "originalName", sanitizeFilename(original)));
	}

	@GetMapping(value = "/{id}")
	public ResponseEntity<Resource> download(@PathVariable String id, @RequestParam String name) throws IOException {
		// Lookup omitted for brevity: resolve by id → random filename, authz checks, etc.
		String safeExt = extractExt(name);
		Path path = SAFE_ROOT.resolve(id + "." + safeExt).normalize();
		if (!path.startsWith(SAFE_ROOT) || !Files.exists(path)) return ResponseEntity.notFound().build();

		byte[] bytes = Files.readAllBytes(path);
		return ResponseEntity.ok()
			.header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + sanitizeFilename(name) + "\"")
			.contentType(MediaType.APPLICATION_OCTET_STREAM)
			.body(new ByteArrayResource(bytes));
	}

	private static String extractExt(String filename) {
		int dot = filename.lastIndexOf('.');
		return dot == -1 ? "" : filename.substring(dot + 1).toLowerCase(Locale.ROOT);
	}

	private static String sanitizeFilename(String name) {
		return name.replace("\"","").replace("\r","").replace("\n","");
	}

	private static boolean magicOk(InputStream in, String ext) throws IOException {
		return switch (ext) {
			case "png" -> looksLikePng(in);
			case "jpg", "jpeg" -> looksLikeJpeg(in);
			case "pdf" -> looksLikePdf(in);
			default -> false;
		};
	}

	private static boolean looksLikePng(InputStream in) throws IOException {
		in.mark(8);
		byte[] sig = in.readNBytes(8);
		in.reset();
		byte[] png = new byte[]{(byte)137,80,78,71,13,10,26,10};
		return Arrays.equals(sig, png);
	}

	private static boolean looksLikeJpeg(InputStream in) throws IOException {
		in.mark(2);
		byte[] sig = in.readNBytes(2);
		in.reset();
		return sig.length == 2 && sig[0] == (byte)0xFF && sig[1] == (byte)0xD8;
	}

	private static boolean looksLikePdf(InputStream in) throws IOException {
		in.mark(5);
		byte[] sig = in.readNBytes(5);
		in.reset();
		// "%PDF-"
		return sig.length == 5 &&
		       sig[0] == '%' && sig[1] == 'P' && sig[2] == 'D' && sig[3] == 'F' && sig[4] == '-';
	}
}
```

Optional ClamAV client

```java
// src/main/java/com/example/upload/[ClamAvScanner.java](http://ClamAvScanner.java)
package com.example.upload;

import org.springframework.stereotype.Component;
import [java.io](http://java.io).*;
import [java.net](http://java.net).Socket;
import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;

@Component
public class ClamAvScanner {
	private final String host = System.getProperty("[clamav.host](http://clamav.host)","[localhost](http://localhost)");
	private final int port = Integer.getInteger("clamav.port", 3310);

	public boolean scan(Path file) throws IOException {
		try (Socket sock = new Socket(host, port);
		     OutputStream out = sock.getOutputStream();
		     InputStream in = sock.getInputStream()) {
			out.write("zINSTREAM\0".getBytes([StandardCharsets.US](http://StandardCharsets.US)_ASCII));
			try (InputStream fin = Files.newInputStream(file)) {
				byte[] buf = new byte[8192];
				int r;
				while ((r = [fin.read](http://fin.read)(buf)) != -1) {
					out.write(ByteBuffer.allocate(4).putInt(r).array());
					out.write(buf, 0, r);
				}
			}
			out.write(new byte[]{0,0,0,0});
			out.flush();
			ByteArrayOutputStream resp = new ByteArrayOutputStream();
			in.transferTo(resp);
			return resp.toString([StandardCharsets.US](http://StandardCharsets.US)_ASCII).contains("OK");
		}
	}
}
```

---

### Tests: traversal, polyglot, oversize, valid case

```java
// src/test/java/com/example/upload/[SafeUploadControllerTest.java](http://SafeUploadControllerTest.java)
package com.example.upload;

import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.mock.web.MockMultipartFile;
import org.springframework.test.web.servlet.MockMvc;

import java.nio.file.*;
import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.multipart;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(SafeUploadController.class)
class SafeUploadControllerTest {

	@Autowired
	MockMvc mvc;

	@MockBean
	ClamAvScanner av;

	static Path tempRoot;

	@BeforeAll
	static void setUpAll() throws Exception {
		tempRoot = Files.createTempDirectory("uploads-test-");
		System.setProperty("upload.dir", tempRoot.toString());
	}

	@AfterAll
	static void tearDownAll() throws Exception {
		System.clearProperty("upload.dir");
		try (var s = Files.list(tempRoot)) {
			s.forEach(p -> { try { Files.deleteIfExists(p); } catch (Exception ignored) {} });
		}
		Files.deleteIfExists(tempRoot);
	}

	@BeforeEach
	void setup() {
		when(av.scan(any())).thenReturn(true);
	}

	@Test
	void rejectsTraversalInFilenameAndStillStoresSafely() throws Exception {
		byte[] png = new byte[]{(byte)137,80,78,71,13,10,26,10, 0,0,0,0};
		MockMultipartFile file = new MockMultipartFile(
			"file",
			"../../../../etc/passwd.png",
			"image/png",
			png
		);

		mvc.perform(multipart("/upload")
				.file(file)
				.contentType(MediaType.MULTIPART_FORM_DATA))
			.andExpect(status().isOk())
			.andExpect(jsonPath("$.id").exists());

		try (var stream = Files.list(tempRoot)) {
			Assertions.assertTrue(stream.findAny().isPresent(), "Expected a saved file in SAFE_ROOT");
		}
	}

	@Test
	void rejectsPolyglotMismatch_ExtJpgMagicPng() throws Exception {
		byte[] jpegHeaderButPngExt = new byte[]{(byte)0xFF,(byte)0xD8, 0x11,0x22,0x33};
		MockMultipartFile file = new MockMultipartFile(
			"file",
			"avatar.png",
			"image/png",
			jpegHeaderButPngExt
		);

		mvc.perform(multipart("/upload").file(file).contentType(MediaType.MULTIPART_FORM_DATA))
			.andExpect(status().isBadRequest())
			.andExpect(jsonPath("$.error").value("Magic bytes mismatch"));
	}

	@Test
	void rejectsOversizeBody() throws Exception {
		int size = 9 * 1024 * 1024; // 9MB > 8MB
		byte[] big = new byte[size];
		big[0] = (byte)137; big[1]=80; big[2]=78; big[3]=71; big[4]=13; big[5]=10; big[6]=26; big[7]=10;

		MockMultipartFile file = new MockMultipartFile(
			"file",
			"large.png",
			"image/png",
			big
		);

		mvc.perform(multipart("/upload").file(file).contentType(MediaType.MULTIPART_FORM_DATA))
			.andExpect(status().isBadRequest())
			.andExpect(jsonPath("$.error").value("Invalid size"));
	}

	@Test
	void acceptsValidPngWithinLimit() throws Exception {
		byte[] png = new byte[]{(byte)137,80,78,71,13,10,26,10, 0,1,2,3,4};
		MockMultipartFile file = new MockMultipartFile(
			"file",
			"ok.png",
			"image/png",
			png
		);

		mvc.perform(multipart("/upload").file(file).contentType(MediaType.MULTIPART_FORM_DATA))
			.andExpect(status().isOk())
			.andExpect(jsonPath("$.id").exists())
			.andExpect(jsonPath("$.originalName").value("ok.png"));
	}
}
```

---

### Notes on how attacks work

- Path traversal: Client tries to smuggle paths via filename. We never use client paths and normalize destination within SAFE_ROOT, so traversal fails
- Polyglot: Misleading extension is rejected by magic-byte sniffing. Blocks .png that’s actually JPEG or a script payload
- Oversize: App-layer size bound rejects large payloads. Combine with server multipart limits for layered protection