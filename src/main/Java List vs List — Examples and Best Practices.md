# Java: List vs List<?> — Examples and Best Practices

## TL;DR

- List is a raw type. Avoid it.
- List<?> is type-safe but write-restricted (you can’t add elements except null). Use it to expose read-only lists of unknown element type.
- Prefer concrete generics like List<String> whenever you know the element type.

---

## Why raw List is dangerous

Raw types disable generics, losing compile-time type checks and causing unchecked warnings and possible ClassCastException at runtime.

```java
List raw = new ArrayList(); // raw type — don’t do this
raw.add("hello");
raw.add(42);

for (Object o : raw) {
	String s = (String) o; // ClassCastException when o is 42
}
```

---

## What List<?> means

A List<?> is a list whose element type is specific but unknown to you. You can read items as Object, but you cannot add any element except null.

```java
List<?> unknowns = List.of("a", "b", "c");
Object first = unknowns.get(0); // OK — read as Object
// unknowns.add("x"); // Compile error
unknowns.add(null); // The only allowed add
```

Why this restriction? Adding a String to a List<?> could break type safety if the actual list is, say, a List<Integer>.

---

## List<?> vs List<Object>

- List<Object> means a list that can hold any object, and you can add any object to it.
- List<?> means a list of some specific type you don’t know; you cannot add items to it.

```java
List<Object> any = new ArrayList<>();
any.add("hello");
any.add(42); // OK

List<?> some = new ArrayList<String>();
// some.add(42); // Compile error
```

---

## Returning lists: which type to choose?

- Return List<T> when you know T and want callers to have full, type-safe access.
- Return List<?> when you want to expose a read-only view without revealing or allowing modification of the element type.
- Avoid returning raw List.

```java
// Best: concrete element type
List<Customer> findCustomers() { ... }

// Read-only exposure when T shouldn’t be exposed
List<?> getInternalBufferView() { return internal; }
```

If you want a read-only API surface but still reveal T, return List<T> and wrap with Collections.unmodifiableList(...).

```java
List<Order> getOrders() {
	return Collections.unmodifiableList(this.orders);
}
```

---

## PECS refresher for wildcards

Producer Extends, Consumer Super.

- ? extends T: use when you only read from the source (it produces T). You still cannot add.
- ? super T: use when you write T into the sink (it consumes T). You can add Ts, but reads are Objects.

```java
void copyAll(List<? extends Number> src, List<? super Number> dst) {
	for (Number n : src) {
		dst.add(n); // OK
	}
}
```

---

## Practical examples

### 1) Iterating a List<?> safely

```java
void printAll(List<?> items) {
	for (Object o : items) {
		System.out.println(o);
	}
}
```

### 2) Accepting any list of strings using upper bound

```java
void processStrings(List<? extends CharSequence> seqs) {
	for (CharSequence s : seqs) {
		// use s
	}
	// seqs.add("x"); // still not allowed
}
```

### 3) Collecting into a consumer using lower bound

```java
void addDefaults(List<? super Integer> sink) {
	sink.add(0);
	sink.add(1);
}
```

### 4) Returning a read-only view with known T

```java
class Repo {
	private final List<String> names = new ArrayList<>();
	public List<String> names() {
		return Collections.unmodifiableList(names);
	}
}
```

---

## Best practices

- Never use raw List. Always parameterize.
- Prefer concrete types: List<T> beats List<?> for public APIs when T is known.
- For parameters that only need to be read, consider ? extends T.
- For parameters you write into, consider ? super T.
- For return types, prefer List<T> or unmodifiable views. Use List<?> only when you must hide T.
- Document mutability: say whether callers can modify the returned list.
- Avoid exposing internal mutable collections directly; return copies or unmodifiable wrappers.

---

## Anti-patterns to avoid

- Returning raw List and casting internally.
- Using List<Object> when you really mean “any list of some specific T”. Use wildcards instead.
- Exposing internal lists that callers can mutate, breaking invariants.

---

## Quick quiz

What’s allowed with List<?>?

- Reading elements as Object: Yes
- Adding non-null elements: No
- Adding null: Yes
- Setting by index with a non-null value: No

Answers: Yes, No, Yes, No.