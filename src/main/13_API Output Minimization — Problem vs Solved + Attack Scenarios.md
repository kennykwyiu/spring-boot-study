# API Output Minimization — Problem vs Solved + Attack Scenarios

### Purpose

Practical “problem vs solved” patterns to minimize API output, plus concrete attacker playbooks that abuse over‑exposed fields to pivot, exfiltrate, or bring down systems.

---

### Quick checklist

- Use DTOs or projections. Never return entities
- Default‑deny fields. Explicit allow‑list only
- Add contract tests to assert exact field sets

---

### 1) Problem vs Solved: Entities vs DTOs

Problem

```java
// Returns Entity directly — leaks internal fields and future changes
@GetMapping("/api/users/{id}")
public User get(@PathVariable Long id) { return repo.findById(id).orElseThrow(); }
```

Solved (DTO)

```java
public record UserDto(Long id, String username, String email) {}

@GetMapping("/api/users/{id}")
public UserDto get(@PathVariable Long id) {
	var u = repo.findById(id).orElseThrow();
	return new UserDto(u.getId(), u.getUsername(), u.getEmail());
}
```

Solved (Projection)

```java
interface UserView { Long getId(); String getUsername(); String getEmail(); }
@GetMapping("/api/users/{id}")
public ResponseEntity<UserView> get(@PathVariable Long id) {
	return repo.findProjectedById(id).map(ResponseEntity::ok).orElseThrow();
}
```

Why better

- Decouples API from persistence
- Small, predictable payloads
- Prevents future field leaks by construction

---

### 2) Problem vs Solved: Default‑deny and allow‑list

Problem

```java
// Anything with a getter may serialize in responses later
class AccountProfile { Long id; String displayName; String email; String internalNote; }
```

Solved A — Minimal DTO

```java
public record AccountProfileDto(Long id, String displayName) {}
```

Solved B — Include list on legacy type

```java
@JsonIncludeProperties({"id","displayName"})
class AccountProfile { Long id; String displayName; String email; String internalNote; }
```

Solved C — Per‑endpoint allow‑list

```java
@JsonFilter("apFilter")
class AccountProfile { Long id; String displayName; String email; String internalNote; }

@GetMapping("/api/profile/{id}")
public MappingJacksonValue get(@PathVariable Long id) {
	var p = svc.get(id);
	var mjv = new MappingJacksonValue(p);
	var fp = new SimpleFilterProvider()
		.addFilter("apFilter", SimpleBeanPropertyFilter.filterOutAllExcept("id","displayName"));
	mjv.setFilters(fp);
	return mjv;
}
```

---

### 3) Contract tests: assert exact field set

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserApiContractTest {
	@Autowired WebTestClient web;
	@Test void getUser_has_exact_fields() {
		Set<String> expected = Set.of("id","username","email");
		var json = web.get().uri("/api/users/{id}", 1)
			.exchange().expectStatus().isOk()
			.expectBody(String.class).returnResult().getResponseBody();
		Set<String> fields = [JsonPath.read](http://JsonPath.read)(json, "$").keySet();
		assertEquals(expected, fields);
	}
}
```

---

### How attackers abuse over‑exposed API output

- Data exfiltration via forgotten fields
    - Scenario: Response includes email, phone, internal flags, feature toggles, or debug tokens.
    - Impact: Privacy breaches, targeted phishing, competitor intel, compliance incidents.
    - Signal: Top‑level keys drift over time as entities evolve.
- Privilege inference and lateral movement
    - Scenario: Hidden roles or ACL lists leak in responses. Adversary maps who has admin or elevated scopes.
    - Impact: Focused credential stuffing or social engineering against high‑value accounts.
    - Pivot: Use leaked IDs to drive IDOR probing on related endpoints.
- Mass assignment and over‑posting coupling
    - Scenario: Clients receive fields that also exist on write models. Attackers reflect these back in POST/PUT to toggle isAdmin, status, or ownerId.
    - Impact: Escalation, object theft, workflow skips. Output bloat makes input attacks easier to guess.
- IDOR amplification
    - Scenario: List endpoints expose sequential IDs and owner IDs. Adversary enumerates and pairs IDs across endpoints to discover accessible objects.
    - Impact: Cross‑tenant data access.
- DoS via object graph explosion
    - Scenario: Serializing full entities with lazy relationships creates huge payloads or N+1 storms.
    - Impact: CPU spikes, memory pressure, network egress costs, client crashes.
- Cache poisoning and stale‑auth leakage
    - Scenario: Responses include user‑specific secrets but are sent with cacheable headers.
    - Impact: Shared or CDN cache leaks PII to other users.
- Feature probing and roadmap disclosure
    - Scenario: Experimental fields, internal toggles, or version banners reveal upcoming features and third‑party vendors.
    - Impact: Attackers look up known vulns in disclosed components.

---

### Red‑team playbook: minimal steps an attacker takes

1. Spider JSON schemas from public endpoints, record all field names and types
2. Correlate IDs and owner fields across endpoints to map relationships
3. Hunt for auth‑relevant booleans or role lists in read responses
4. Try reflective over‑posting of these fields on write endpoints
5. Abuse high‑fan‑out lists to cause server‑side joins and large payloads (DoS)
6. Probe caching headers on personalized responses; attempt shared cache fetch

---

### Defensive patterns that stop the above

- Enforce DTOs or projections everywhere. Ban Entity in controller signatures
- Explicit allow‑list per endpoint. New fields require code review
- Contract tests in CI for every public endpoint
- Pagination and field selection caps. Reject ?expand=... beyond allow‑list
- Separate read and write models. Ignore unexpected fields on write, and fail on unknowns for inbound
- Add response size budgets with server and gateway limits
- Privacy review for any user‑specific data. No secrets or tokens in GET responses
- Cache headers: private, short TTL, or no‑store for personalized content

---

### Before vs After: end‑to‑end snippet

Before

```java
@GetMapping("/api/orders/{id}")
public Order get(@PathVariable Long id) { return repo.findById(id).orElseThrow(); }
```

After

```java
public record OrderDto(Long id, String number, String status, BigDecimal total) {}
@GetMapping("/api/orders/{id}")
public OrderDto get(@PathVariable Long id) {
	var o = repo.findById(id).orElseThrow();
	return new OrderDto(o.getId(), o.getNumber(), o.getStatus(), o.getTotal());
}
```

Contract test

```java
@Test void order_contract_exact_fields() {
	Set<String> expected = Set.of("id","number","status","total");
	// fetch and assert key set equality as above
}
```

---

### Rollout plan

- Phase 1: Identify endpoints returning entities. Add DTOs for top 5
- Phase 2: Introduce projection queries for heavy lists. Add pagination caps
- Phase 3: Add contract tests. Block merges on field drift
- Phase 4: Add cache policy checks and response size budgets

---

### Notes

- Keep DTOs in an api.dto package to avoid accidental reuse on writes unless intended
- Consider a field‑mask layer only if it is backed by a strict allow‑list, never free‑form