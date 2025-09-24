# Spring Boot 3.4.5 — Logging & Error Handling: Problem vs Best Practice

### Overview

Code pairs showing vulnerable “Problem” examples and safer “Best practice” implementations for:

- Log Forging
- Stored Log Forging
- Improper Exception Handling

---

## 1) Log Forging

Problem

```java
@RestController
class ColorController {
	private static final Logger log = LoggerFactory.getLogger(ColorController.class);

	@PostMapping("/colors")
	ResponseEntity<Void> pick(@RequestParam String color) {
		// Attacker can inject CR/LF to forge extra log lines
		[log.info](http://log.info)("Color was picked: {}", color);
		return ResponseEntity.accepted().build();
	}
}
```

Best practice

```java
@RestController
class ColorController {
	private static final Logger log = LoggerFactory.getLogger(ColorController.class);

	@PostMapping("/colors")
	ResponseEntity<Void> pick(@RequestParam String color) {
		String safe = sanitizeForLogs(color);
		[log.info](http://log.info)("Color was picked: {}", safe);
		return ResponseEntity.accepted().build();
	}

	private static String sanitizeForLogs(String s) {
		if (s == null) return "";
		return s.replace('\n', '_').replace('\r', '_').replace('\t', '_');
	}
}
```

Defense-in-depth: Logback CR/LF stripping

```xml
<!-- src/main/resources/logback-spring.xml -->
<configuration>
	<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
		<encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
			<pattern>%d{ISO8601} %-5level [%thread] %logger{36} - %replace(%msg){'\r|\n','_'} cid=%X{cid}%n</pattern>
		</encoder>
	</appender>
	<root level="INFO"><appender-ref ref="CONSOLE"/></root>
</configuration>
```

Per-request correlation ID

```java
@Component
class CorrelationIdFilter extends OncePerRequestFilter {
	@Override
	protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		String cid = Optional.ofNullable(req.getHeader("X-Correlation-Id"))
			.filter(v -> v.length() <= 64)
			.orElse(UUID.randomUUID().toString());
		MDC.put("cid", cid);
		try { chain.doFilter(req, res); } finally { MDC.remove("cid"); }
	}
}
```

---

## 2) Stored Log Forging

Problem

```java
@Service
class UserService {
	private static final Logger log = LoggerFactory.getLogger(UserService.class);
	private final JdbcTemplate jdbc;
	UserService(JdbcTemplate jdbc) { this.jdbc = jdbc; }

	void parseUserIdAndLog() {
		String userIdRaw = jdbc.queryForObject("select user_id from t where ...", String.class);
		try {
			Integer.parseInt(userIdRaw);
		} catch (NumberFormatException nfe) {
			// userIdRaw might contain "\nINFO admin logged in"
			log.error("UserId {} parse error: {}", userIdRaw, nfe.getMessage());
		}
	}
}
```

Best practice

```java
@Service
class UserService {
	private static final Logger log = LoggerFactory.getLogger(UserService.class);
	private final JdbcTemplate jdbc;
	UserService(JdbcTemplate jdbc) { this.jdbc = jdbc; }

	void parseUserIdAndLog() {
		String userIdRaw = jdbc.queryForObject("select user_id from t where ...", String.class);
		String safe = sanitizeForLogs(userIdRaw);
		try {
			Integer.parseInt(userIdRaw);
		} catch (NumberFormatException nfe) {
			log.error("UserId {} parse error: {}", safe, nfe.toString());
		}
	}

	private static String sanitizeForLogs(String s) {
		if (s == null) return "";
		return s.replace('\n', '_').replace('\r', '_').replace('\t', '_');
	}
}
```

Schema tips

- Constrain length and charset in DB columns.
- Validate on write, sanitize on log emission.

---

## 3) Improper Exception Handling

Problem: leaks stack traces or sensitive details

```java
@RestController
class OrdersController {
	private final OrdersService svc;
	OrdersController(OrdersService svc) { this.svc = svc; }

	@GetMapping("/orders/{id}")
	Order get(@PathVariable long id) throws SQLException {
		try {
			return svc.load(id);
		} catch (SQLException ex) {
			// Sends internals to the client or causes crash
			throw new RuntimeException("DB error: " + ex.getMessage(), ex);
		}
	}
}
```

Best practice: ProblemDetail + global advice + safe config

application.yml

```yaml
server:
  error:
    include-stacktrace: never
    include-message: never
    include-binding-errors: never
    whitelabel:
      enabled: false
spring:
  mvc:
    problemdetails:
      enabled: true
```

Global handler

```java
@RestControllerAdvice
class GlobalExceptionHandler {
	private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

	@ExceptionHandler(MethodArgumentNotValidException.class)
	ResponseEntity<ProblemDetail> handleValidation(MethodArgumentNotValidException ex, HttpServletRequest req) {
		ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
		pd.setTitle("Validation failed");
		pd.setDetail("One or more fields are invalid.");
		pd.setProperty("correlationId", MDC.get("cid"));
		log.warn("400 validation cid={} path={} err={}", MDC.get("cid"), req.getRequestURI(), ex.toString());
		return ResponseEntity.badRequest().body(pd);
	}

	@ExceptionHandler({ DataAccessException.class, SQLException.class })
	ResponseEntity<ProblemDetail> handleData(Exception ex, HttpServletRequest req) {
		ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
		pd.setTitle("Database error");
		pd.setDetail("Unexpected persistence error.");
		pd.setProperty("correlationId", MDC.get("cid"));
		log.error("500 db cid={} path={} err={}", MDC.get("cid"), req.getRequestURI(), ex.toString());
		return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(pd);
	}

	@ExceptionHandler(Exception.class)
	ResponseEntity<ProblemDetail> handleAny(Exception ex, HttpServletRequest req) {
		ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
		pd.setTitle("Unexpected error");
		pd.setDetail("Please contact support with the correlation ID.");
		pd.setProperty("correlationId", MDC.get("cid"));
		log.error("500 unhandled cid={} path={} err={}", MDC.get("cid"), req.getRequestURI(), ex.toString());
		return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(pd);
	}
}
```

Controller using generic errors only

```java
@RestController
class OrdersController {
	private final OrdersService svc;
	OrdersController(OrdersService svc) { this.svc = svc; }

	@GetMapping("/orders/{id}")
	Order get(@PathVariable long id) {
		return svc.load(id); // exceptions are translated by @RestControllerAdvice
	}
}
```

Access log pattern without sensitive content

```yaml
server:
  tomcat:
    accesslog:
      enabled: true
      pattern: '%t %a "%r" %s %b cid=%{cid}r'
```

---

### Notes

- Always sanitize untrusted data at the log boundary, even if validated earlier.
- Prefer structured logging and avoid logging bodies, headers, or secrets.
- Use correlation IDs to tie server logs to user-visible errors without exposing stack traces.