# Concurrency Issues — Problems and Better Examples

> Practical before-and-after examples with concise explanations for concurrency risks in section “5. Concurrency Issues.”
> 

### Overview

- Focus: Race condition format flaw. Server DoS by loop. Unsynchronized access to shared data.
- Goals: Understand why each happens. How it fails. How to fix with minimal, correct patterns.

---

### A) Race condition format flaw

Problem: Sharing a non-thread-safe formatter across threads corrupts internal mutable state, producing wrong or malformed output.

Bad — shared, non-thread-safe formatter:

```java
import java.text.NumberFormat;
import java.util.Locale;

public class MoneyFormatRace {
	static final NumberFormat NF = NumberFormat.getCurrencyInstance([Locale.US](http://Locale.US));

	public static void main(String[] args) throws InterruptedException {
		Runnable r1 = () -> System.out.println(NF.format(100.0));
		Runnable r2 = () -> System.out.println(NF.format(50.0));
		Thread t1 = new Thread(r1);
		Thread t2 = new Thread(r2);
		t1.start();
		t2.start();
		t1.join();
		t2.join();
	}
}
```

Better — per-call instance (no shared state):

```java
import java.text.NumberFormat;
import java.util.Locale;

public class MoneyFormatSafe {
	public static void main(String[] args) throws InterruptedException {
		Runnable r1 = () -> {
			NumberFormat nf = NumberFormat.getCurrencyInstance([Locale.US](http://Locale.US));
			System.out.println(nf.format(100.0));
		};
		Runnable r2 = () -> {
			NumberFormat nf = NumberFormat.getCurrencyInstance([Locale.US](http://Locale.US));
			System.out.println(nf.format(50.0));
		};
		Thread t1 = new Thread(r1);
		Thread t2 = new Thread(r2);
		t1.start();
		t2.start();
		t1.join();
		t2.join();
	}
}
```

Also good — per-thread reuse via ThreadLocal:

```java
static final ThreadLocal<NumberFormat> NF_TL =
	ThreadLocal.withInitial(() -> NumberFormat.getCurrencyInstance([Locale.US](http://Locale.US)));

String s = NF_TL.get().format(100.0);
```

Key guidance

- Do not share non-thread-safe utilities.
- Use per-call or per-thread instances. Prefer immutable, stateless APIs.

---

### B) Server DoS by loop

Problem: An endpoint uses user-controlled values to size work. A large value ties up CPU or resources, starving other requests.

Bad — unbounded loop from user input:

```jsx
// Express.js handler
app.get("/work", (req, res) => {
	const iterations = Number(req.query.iterations); // e.g., 10_000_000_000
	for (let i = 0; i < iterations; i++) {
		// expensive work
	}
	res.send("done");
});
```

Better — validate, cap, and fail fast:

```jsx
const MAX_ITERATIONS = 100000; // tune per capacity

app.get("/work", (req, res) => {
	const n = Number(req.query.iterations);
	if (!Number.isFinite(n) || n < 0) return res.status(400).send("invalid iterations");
	if (n > MAX_ITERATIONS) return res.status(429).send("too many iterations");

	let sum = 0;
	for (let i = 0; i < n; i++) sum += i;
	res.json({ ok: true, sum });
});
```

Stronger hardening

- Add per-request timeout or abort signal.
- Rate-limit the endpoint.
- Move heavy work to a job queue with worker concurrency limits.
- Bound batch sizes, pagination, graph depth, and recursion.

---

### C) Unsynchronized access to shared data

Problem: Multiple threads update shared state without atomicity. Increments are lost. Invariants break. In web apps, singletons leak state across requests.

Bad — racy increment:

```java
public class Counter {
	static int count = 0;

	public static void main(String[] args) throws InterruptedException {
		Thread t1 = new Thread(() -> { for (int i = 0; i < 1_000_000; i++) count++; });
		Thread t2 = new Thread(() -> { for (int i = 0; i < 1_000_000; i++) count++; });
		t1.start();
		t2.start();
		t1.join();
		t2.join();
		System.out.println("count = " + count); // often < 2_000_000
	}
}
```

Better — atomic operations:

```java
import java.util.concurrent.atomic.AtomicInteger;

public class CounterSafe {
	static final AtomicInteger count = new AtomicInteger();

	public static void main(String[] args) throws InterruptedException {
		Thread t1 = new Thread(() -> { for (int i = 0; i < 1_000_000; i++) count.incrementAndGet(); });
		Thread t2 = new Thread(() -> { for (int i = 0; i < 1_000_000; i++) count.incrementAndGet(); });
		t1.start();
		t2.start();
		t1.join();
		t2.join();
		System.out.println("count = " + count.get()); // exactly 2_000_000
	}
}
```

Alternative — lock for compound invariants:

```java
public class CounterLocked {
	static int count = 0;
	static final Object lock = new Object();

	public static void inc() {
		synchronized (lock) {
			count++;
		}
	}
}
```

Servlet gotcha — avoid shared mutable fields in singletons:

Bad:

```java
public class ShopServlet extends HttpServlet {
	private User currentUser; // shared across requests — leaks between users!

	protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
		currentUser = (User) req.getSession().getAttribute("user"); // unsafe
		// ...
	}
}
```

Better:

```java
public class ShopServlet extends HttpServlet {
	protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
		User user = (User) req.getSession().getAttribute("user"); // request-scoped
		// ...
	}
}
```

Guidance

- Use Atomic* for counters and flags.
- Use locks for multi-field consistency.
- Keep mutable state request-scoped or thread-confined.
- Prefer immutable data and message-passing.

---

### Quick checklist

- [ ]  No shared non-thread-safe utilities. Use per-call or ThreadLocal.
- [ ]  Validate and cap all user-influenced loop/work sizes. Add timeouts, rate limits.
- [ ]  Shared state updated atomically. Use Atomic*, locks, or immutable snapshots.
- [ ]  No request-specific fields on singletons (servlets, Spring singletons).
- [ ]  Load test with concurrency to reveal interleavings and long-tail latencies.

---

### References in workspace

- See Risk Cat categorization → 5. Concurrency Issues for original entries.

---

### Deeper explanations and operational guidance

### Race condition format flaw — details

- Why it happens
    - Many formatters keep internal buffers and locale state. Concurrent calls interleave these buffers.
- Detection strategies
    - Load tests with mixed values. Fuzzy testing around formatters, parsers, encoders.
    - Static analysis: flags on shared singletons of non-thread-safe types.
- Remediation patterns
    - Per-call construction for cheap objects. ThreadLocal for expensive objects. Prefer stateless, pure functions when available.
- Tradeoffs
    - Per-call creation increases allocations. ThreadLocal reduces allocations but must be cleared in thread pools if objects hold large memory.

### Server DoS by loop — details

- Typical sources
    - iteration, limit, depth, pageSize, batchSize, recursion depth, regex complexity, JSON size, compression level, image resize dimensions.
- Defense-in-depth
    - Input validation and caps. Per-request deadline. Rate limiting and client quotas. Circuit breakers. Queue with bounded workers and backpressure.
- Observability
    - Expose metrics: p95 and p99 latencies, queued jobs, worker saturation, rejections, timeouts. Add structured logs with request IDs.
- Testing
    - Adversarial tests sending extreme values. Chaos tests combining big loops with downstream latency to expose amplification.

### Unsynchronized access to shared data — details

- Failure modes
    - Lost updates, torn reads, invariants broken across multi-field updates, stale visibility without proper memory barriers.
- Java memory model essentials
    - AtomicInteger provides atomic RMW plus happens-before on get after increment.
    - volatile ensures visibility but not atomicity for compound ops.
    - synchronized and locks provide mutual exclusion and ordering.
- Choosing tools
    - Atomic types for simple counters. Locks for multi-field invariants. Immutable snapshots with AtomicReference for configuration hot-swaps.
- Web singleton pitfalls
    - Servlets and many Spring beans are singletons. Keep request state in locals or request/session scope. Prefer stateless services.

---

### Additional concurrency topics

### Deadlocks

- Cause
    - Cyclic lock acquisition, mixed lock orders, calling out to unknown code while holding locks, blocking on futures inside synchronized sections.
- Prevention
    - Establish a global lock order. Minimize lock scope. Avoid blocking calls while holding locks. Prefer tryLock with timeouts and detect contention.
- Example

```java
Lock a = new ReentrantLock();
Lock b = new ReentrantLock();

// Risky: opposite ordering can deadlock
void f1() { a.lock(); try { b.lock(); try { work(); } finally { b.unlock(); } } finally { a.unlock(); } }
void f2() { b.lock(); try { a.lock(); try { work(); } finally { a.unlock(); } } finally { b.unlock(); } }

// Safer: consistent ordering
void fSafe() { acquireInOrder(a, b); try { work(); } finally { releaseInReverse(b, a); } }
```

### Thread pool sizing and saturation

- Guidance
    - Separate pools by workload type: CPU-bound vs I/O-bound. Keep CPU-bound pool near number of cores. I/O-bound can be larger but bounded.
    - Set queue bounds and a rejection policy. Expose metrics and alarms on queue length and execution time.
- Pitfalls
    - Unbounded pools or queues hide saturation then fail catastrophically. Blocking tasks inside common pools (e.g., ForkJoinPool) cause starvation.

### Futures/promises and async pitfalls

- Risks
    - Blocking get() on request threads. Deadlocks when waiting within synchronized code. Lost exceptions.
- Patterns
    - Prefer async composition (thenApply, thenCompose). Use timeouts on joins. Propagate cancellation. Avoid blocking in event loops.

### Collections and publication safety

- Use concurrent collections (ConcurrentHashMap, CopyOnWriteArrayList) where appropriate.
- Safely publish shared objects via final fields, volatile refs, or immutable value objects.

---

### Practical checklist (ops-ready)

- [ ]  Caps on all user-driven sizes and depths. Server-side validation only.
- [ ]  Per-request deadlines and timeouts. Abort long handlers.
- [ ]  Rate limits and bounded queues with backpressure.
- [ ]  No shared non-thread-safe utilities. ThreadLocal or per-call.
- [ ]  Atomic or locked updates for all shared mutation. Immutable where possible.
- [ ]  Global lock ordering documented. No blocking while holding locks.
- [ ]  Separate, bounded thread pools. Monitor saturation and rejections.
- [ ]  Load tests that mix concurrency with adversarial inputs.