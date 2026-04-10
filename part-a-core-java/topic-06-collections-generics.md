# Topic 6 — Collections Framework, Generics & HashMap Deep-Dive

> **Format:** Concept → Simple Explanation → How It Works → Code → Real Usage → Interview Answer → Related Concepts
---

## Part 1: Collections Framework Foundation

## Q20. The Three Root Interfaces — List vs Set vs Map

### Concept
The Java Collections Framework provides three root interfaces, each solving a different data storage problem with distinct semantics and performance characteristics.

### Simple Explanation
- **List** = an ordered queue (cinema line) — people have positions, duplicates allowed, access by position  
- **Set** = a bag of unique coins — no duplicates, position irrelevant, fast "is this coin here?" check
- **Map** = a dictionary — keys point to values, keys are unique, fast key-based lookup

### Core Differences & Usage

| Aspect | `List<E>` | `Set<E>` | `Map<K,V>` |
|--------|-----------|---------|-----------|
| **Duplicates** | ✅ Allowed | ❌ No duplicates | ❌ No duplicate keys |
| **Order** | ✅ Insertion order | Depends on impl | Depends on impl |
| **Access** | By index: `get(0)` | By equality: `contains()` | By key: `get(key)` |
| **Null** | Multiple nulls allowed | One null (HashSet) | One null key (HashMap) |
| **Use Case** | Sequences, batches, collections | Uniqueness, membership | Key-value associations |

### Code — Real Usage Examples

```java
// ─── LIST: Ordered, index-based, allows duplicates ────────────────────
List<String> list = new ArrayList<>(List.of("a", "b", "a", "c"));
list.get(0);           // "a" — O(1) direct access by index
list.add("d");         // O(1) amortised append at end
list.add(1, "x");      // O(n) insert at index 1 — shifts everything right
list.remove(0);        // O(n) remove by index — shifts remaining elements
list.contains("a");    // O(n) — must scan entire list

// Common operations
for (int i = 0; i < list.size(); i++) {
    String item = list.get(i);  // Index-based iteration
}
list.forEach(item -> System.out.println(item));  // Functional iteration
List<String> uppercase = list.stream().map(String::toUpperCase).toList();

// ─── SET: Unique elements, no index access ─────────────────────────────
Set<String> set = new HashSet<>(Set.of("apple", "banana", "apple", "cherry"));
// HashSet automatically removes duplicate "apple"
System.out.println(set.size());           // 3 — duplicate dropped
set.contains("apple");                    // O(1) — hash-based lookup
set.add("cherry");                        // False — already in set
set.remove("banana");                     // O(1) removal

// No index access — Sets don't have get(index)
// set.get(0);  ← COMPILE ERROR

// Set operations (union, intersection, difference)
Set<String> set1 = new HashSet<>(Set.of("a", "b", "c"));
Set<String> set2 = new HashSet<>(Set.of("b", "c", "d"));
set1.addAll(set2);                        // Union: {a,b,c,d}
set1.retainAll(set2);                     // Intersection: {b,c}
set1.removeAll(set2);                     // Difference: {a}

// ─── MAP: Key-value associations, unique keys ──────────────────────────
Map<String, Integer> map = new HashMap<>();
map.put("alice", 30);
map.put("bob", 25);
map.put("alice", 31);                     // Overwrites previous value
map.get("alice");                         // 31 — O(1) key-based lookup
map.containsKey("alice");                 // true — O(1)
map.containsValue(25);                    // true — O(n) must scan all values
map.remove("bob");                        // O(1) removal

// Iteration patterns
for (String key : map.keySet()) {
    Integer value = map.get(key);         // O(1) but two lookups per entry!
}
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    String key = entry.getKey();
    Integer value = entry.getValue();     // One-pass, no double lookup ✓
}
map.forEach((key, value) -> System.out.println(key + "=" + value));
```

### Hierarchy & Implementations Cheat Sheet

```
Collection<E>
├── List<E>
│   ├── ArrayList<E>          ✓ Most common, O(1) access
│   ├── LinkedList<E>         O(n) access, O(1) head/tail (but ArrayDeque is better)
│   ├── CopyOnWriteArrayList<E>  Thread-safe, read-heavy, expensive writes
│   └── Vector<E>             Legacy, synchronized (avoid)
│
├── Set<E>
│   ├── HashSet<E>            ✓ Most common, O(1) operations, no order
│   ├── LinkedHashSet<E>      O(1) ops, insertion order
│   ├── TreeSet<E>            O(log n) ops, sorted by Comparable
│   └── EnumSet<E>            O(1) ops, keys are enums (most compact)
│
└── Queue<E>
    ├── ArrayDeque<E> (also Deque)  ✓ Best queue, O(1) head/tail
    ├── LinkedList<E> (also Deque)  Implements Deque but ArrayDeque is faster
    └── PriorityQueue<E>            O(log n), heap-ordered (min-heap by default)

Map<K,V>
├── HashMap<K,V>              ✓ Most common, O(1) average, no order
├── LinkedHashMap<K,V>        O(1), insertion or access-order iteration
├── TreeMap<K,V>              O(log n), sorted by key
├── Hashtable<K,V>            Legacy, synchronized (avoid)
└── ConcurrentHashMap<K,V>    ✓ Thread-safe for concurrent access, O(1) with fine-grained locking
```

### Interview Answer
> "List is the go-to collection when you need a sequence with positional access — ArrayList. Set guarantees uniqueness — choose HashSet for O(1) operations, TreeSet when you need sorted order. Map associates keys to values — HashMap for O(1) general use, LinkedHashMap when insertion order matters, TreeMap when keys need to be sorted, ConcurrentHashMap in multi-threaded contexts.
>
> Common mistakes: choosing LinkedList over ArrayList (ArrayList is faster in nearly all cases due to cache locality), putting non-hashable objects in HashSet (need correct `hashCode` + `equals`), using HashMap with null keys/values and then expecting `get()` null to distinguish 'key not found' vs 'value is null'.
>
> Performance rule: List get is O(1) for ArrayList, O(n) for LinkedList. Set/Map operations are O(1) for Hash-based (HashSet, HashMap, ConcurrentHashMap), O(log n) for Tree-based (TreeSet, TreeMap)."
>
> *Likely follow-up: "What is the difference between ArrayList and LinkedList in terms of when to use each?"*

---

## Part 2: List Implementations — Detailed

## Q20a. ArrayList vs LinkedList — When to Use Each

### Concept & Performance Comparison

```java
// ─── ARRAYLIST: Resizable array ────────────────────────────────────────
List<String> arrayList = new ArrayList<>();  // Initial capacity = 10
// Grows: 10 → 15 → 22 → 33 → ... (1.5× factor, amortised O(1) append)

// Performance:
// get(i):     O(1) — direct array indexing, CPU cache-friendly
// add():      O(1) amortised — append at end
// add(i, e):  O(n) — shifts all elements from i forward
// remove(i):  O(n) — shifts all elements after i backward
// contains(): O(n) — must scan entire list

List<String> al = new ArrayList<>(List.of("a", "b", "c"));
al.get(0);        // "a" — direct array[0] access ✓ O(1)
al.add("d");      // append O(1)
al.add(1, "x");   // O(n) — shifts "b", "c" right

// ─── LINKEDLIST: Doubly-linked list ────────────────────────────────────
List<String> linkedList = new LinkedList<>();  // Each node has prev/next pointers

// Performance:
// get(i):     O(n) — walk from head, worst case n/2 average
// add():      O(1) amortised — append at end (added to tail)
// add(i, e):  O(1) at known node, O(n) to find node i
// remove(i):  O(1) at known node, O(n) to find it
// contains(): O(n) — must scan entire list
// Memory:     ~3× overhead vs ArrayList (prev/next/value pointers)

LinkedList<String> ll = new LinkedList<>(List.of("a", "b", "c"));
ll.get(0);        // O(n) — walk from head ✗ slow
ll.add("d");      // O(1) append ✓
ll.addFirst("z"); // O(1) insert at head ✓  ← LinkedList only implements Deque
ll.addLast("y");  // O(1)
ll.removeFirst(); // O(1)
// But head/tail operations on ArrayList would be O(n) — this is LinkedList's advantage

// ─── WHEN TO USE EACH ──────────────────────────────────────────────────

// ✅ ArrayList is the default:
// → Sequential access (for loops, iteration)
// → Frequent get(index)
// → Append-heavy workloads
// → Memory compact, cache-friendly
List<User> users = new ArrayList<>();

// ❌ Avoid LinkedList for:
// → Random access — it's O(n) and pointer-chasing kills CPU cache
// → General-purpose sequences — ArrayList is just better

// ✅ Use LinkedList ONLY when:
// → Frequently inserting/removing at head/tail (implement custom Deque)
// → But even then, ArrayDeque is usually faster! (cache locality, no pointer dereference)

// Better than LinkedList for queues:
Deque<String> deque = new ArrayDeque<>();  // Array-backed, faster iteration
deque.addFirst("a");   // O(1)
deque.addLast("b");    // O(1)
deque.removeFirst();   // O(1)
deque.removeLast();    // O(1)
```

### Real Project Usage

```java
// ArrayList: the workhorse in almost every project
@Service
public class OrderService {
    public List<Order> getOrdersByUser(Long userId) {
        return orders.stream()
            .filter(o -> o.getUserId().equals(userId))
            .toList();           // Returns ArrayList under the hood
    }
    
    public List<Order> getOrdersByUserWithCustomList(Long userId) {
        List<Order> result = new ArrayList<>();  // Explicit, capacity hint possible
        for (Order o : orders) {
            if (o.getUserId().equals(userId)) {
                result.add(o);
            }
        }
        return result;
    }
}

// ArrayDeque: correct implementation for queues/deques
@Service
public class TaskProcessor {
    private final Deque<Task> taskQueue = new ArrayDeque<>();  ← Not LinkedList!
    
    public void enqueue(Task task) {
        taskQueue.addLast(task);   // O(1)
    }
    
    public Task dequeue() {
        return taskQueue.removeFirst();  // O(1)
    }
}
```

### Why ArrayDeque Beats LinkedList for Queues

```
LinkedList queue operation:
  addLast() → create Node(e, tail.prev, tail) → update pointers → O(1) but pointer chasing
  removeFirst() → head.next → unlink → follow reference chain
  Iteration: node1.next → node2.next → ... (cache misses on each dereference)

ArrayDeque queue operation:
  addLast() → array[tail++] → O(1), memory adjacent
  removeFirst() → return array[head++] → O(1), no pointer chasing
  Iteration: array[i]++ → all in contiguous memory (CPU cache prefetch wins)

Result: ArrayDeque 2-3× faster for queue operations than LinkedList
```

### Interview Answer
> "ArrayList is the default collection for nearly everything — random access is O(1), append is O(1) amortised, and CPU cache locality is friendly. LinkedList is rarely justified. It's O(n) to access an element in the middle, and even for head/tail operations, ArrayDeque is faster due to better cache locality. I've never regretted using ArrayList. I've regretted LinkedList choices multiple times.
>
> The ONLY case where LinkedList might make sense is when you're doing heavy middle insertion/deletion on a known node, but that's vanishingly rare. Use ArrayList or ArrayDeque. If performance metrics show a problem, profile first before blaming the List type."
>
> *Likely follow-up: "What is the Big-O complexity of random access in LinkedList and why?"*

---

## Q21. Set Implementations — HashSet, LinkedHashSet, TreeSet, EnumSet

### Concept & Usage

```java
// ─── HASHSET: Unique elements, no order ────────────────────────────────
// Fastest when order doesn't matter
Set<String> hashSet = new HashSet<>(Set.of("apple", "banana", "cherry"));
hashSet.contains("apple");    // O(1)
hashSet.add("apple");         // False — already there
hashSet.add("date");          // True — new
// Iteration order: arbitrary, depends on hash

for (String fruit : hashSet) {
    System.out.println(fruit);  // Order: unpredictable
}

// ─── LINKEDHASHSET: Unique elements, insertion order ───────────────────
// Fast (O(1)) but predictable iteration
Set<String> linkedSet = new LinkedHashSet<>(Set.of("apple", "banana", "cherry"));
linkedSet.add("date");
for (String fruit : linkedSet) {
    System.out.println(fruit);  // Order: apple, banana, cherry, date ✓ insertion order
}

// Use case: de-duplication while preserving order
List<Integer> numbers = List.of(1, 2, 2, 3, 3, 3, 4);
List<Integer> unique = new ArrayList<>(new LinkedHashSet<>(numbers));
System.out.println(unique);  // [1, 2, 3, 4] — in order, no duplicates

// ─── TREESET: Unique elements, sorted order ───────────────────────────
// O(log n) operations but always sorted
Set<String> treeSet = new TreeSet<>(Set.of("cherry", "apple", "banana"));
treeSet.add("date");
for (String fruit : treeSet) {
    System.out.println(fruit);  // Order: apple, banana, cherry, date ✓ sorted
}

// TreeSet operations:
treeSet.first();      // "apple"
treeSet.last();       // "date"
treeSet.headSet("c"); // {"apple", "banana"} — all < "c"
treeSet.tailSet("c"); // {"cherry", "date"} — all ≥ "c"
treeSet.subSet("b", "d");  // {"banana", "cherry"} — [b, d)

// Custom ordering via Comparator
Set<Employee> empByName = new TreeSet<>(Comparator.comparing(Employee::getName));
empByName.add(new Employee("Alice", 30));
empByName.add(new Employee("Bob", 25));
// Always sorted by name

// ─── ENUMSET: Unique enums, ultra-compact ─────────────────────────────
// Backed by a bitset — fastest and most memory-efficient for enum keys
public enum Permission { READ, WRITE, EXECUTE, DELETE }

Set<Permission> perms = EnumSet.of(Permission.READ, Permission.WRITE);
perms.add(Permission.EXECUTE);   // O(1)
perms.contains(Permission.READ);  // O(1), super fast (bit check)

Set<Permission> allPerms = EnumSet.allOf(Permission.class);  // All 4 permissions
// Iteration: guaranteed order (ordinal order)

// Use in practice: flags, capabilities, group roles
Set<Permission> userPerms = EnumSet.of(Permission.READ, Permission.EXECUTE);
if (userPerms.contains(Permission.WRITE)) { /* deny */ }
```

### Set Implementation Comparison

| Aspect | `HashSet` | `LinkedHashSet` | `TreeSet` | `EnumSet` |
|--------|-----------|-----------------|-----------|-----------|
| **Order** | Arbitrary | Insertion | Sorted | Ordinal |
| **add/remove/contains** | O(1) | O(1) | O(log n) | O(1) |
| **Iteration** | O(n) unordered | O(n) ordered | O(n) ordered | O(n) ordered |
| **Memory** | Compact | ~1.2× HashSet | ~1.5× HashSet | Bitset (tiny!) |
| **Use Case** | General, no order | Preserve input order | Sorted results | Enum flags |

### Interview Answer
> "HashSet is the default — O(1) operations, no overhead. Use LinkedHashSet when insertion order matters (de-duplication while preserving sequence). TreeSet when you need sorted iteration or range queries (headSet, tailSet, subSet). EnumSet when keys are enums — incredible space efficiency (backed by bitset) and O(1) operations.
>
> Most common mistake: using TreeSet or LinkedHashSet when HashSet works — unnecessary O(log n) or insertion overhead."
>
> *Likely follow-up: "How does EnumSet achieve O(1) operations?"*

---

## Part 3: Map Implementations — Deep Dive

## Q21b. HashMap, LinkedHashMap, TreeMap — Comparison & Usage

### Concept

Three fundamental Map implementations with different ordering and performance characteristics.

### Code — Side-by-side Comparison

```java
// ─── HASHMAP: Unordered, fastest ──────────────────────────────────────
Map<String, Integer> hashMap = new HashMap<>();
hashMap.put("banana", 2);
hashMap.put("apple", 1);
hashMap.put("cherry", 3);
hashMap.forEach((k, v) -> System.out.print(k + " "));
// Output: apple cherry banana  (arbitrary order)

// ─── LINKEDHASHMAP: Insertion order, still fast ───────────────────────
// Adds a doubly-linked list across entries to track insertion
Map<String, Integer> linkedMap = new LinkedHashMap<>();
linkedMap.put("banana", 2);
linkedMap.put("apple", 1);
linkedMap.put("cherry", 3);
linkedMap.forEach((k, v) -> System.out.print(k + " "));
// Output: banana apple cherry  (exactly insertion order)

// ─── TREEMAP: Sorted by key, slower but ordered ───────────────────────
// Backed by Red-Black tree
Map<String, Integer> treeMap = new TreeMap<>();
treeMap.put("banana", 2);
treeMap.put("apple", 1);
treeMap.put("cherry", 3);
treeMap.forEach((k, v) -> System.out.print(k + " "));
// Output: apple banana cherry  (sorted alphabetically)

// TreeMap navigation operations (unique to SortedMap):
treeMap.firstKey();         // "apple"
treeMap.lastKey();          // "cherry"
treeMap.floorKey("b");      // apple (highest key ≤ "b")
treeMap.ceilingKey("b");    // banana (lowest key ≥ "b")
treeMap.headMap("c");       // {apple, banana} — all < "c"
treeMap.tailMap("c");       // {cherry} — all ≥ "c"
treeMap.subMap("a", "c");   // {apple, banana} — [a, c)
```

### LinkedHashMap as LRU Cache Base

```java
// Access-order LinkedHashMap: frequently accessed items move to end
// Perfect base for building an LRU (Least Recently Used) cache

Map<String, Integer> lruCache = new LinkedHashMap<String, Integer>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Integer> eldest) {
        return size() > 3;   // Keep max 3 entries; evict LRU when exceeded
    }
};

// Simulate cache with access-based eviction
lruCache.put("a", 1); // [a]
lruCache.put("b", 2); // [a, b]
lruCache.put("c", 3); // [a, b, c]
lruCache.get("a");    // Access "a" — moves to end: [b, c, a]
lruCache.put("d", 4); // Size exceeds 3 → evict eldest (now "b"): [c, a, d]
System.out.println(lruCache.keySet()); // [c, a, d]

// Real production LRU cache:
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(16, 0.75f, true);  // access-order mode
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}

LRUCache<String, String> cache = new LRUCache<>(100);
cache.put("user:1", "Alice");
String user = cache.get("user:1");  // Moves to end (most recently used)
```

### EnumMap — Specialized Map for Enum Keys

```java
// EnumMap: Array-backed map where keys are enum constants
// Performance: O(1) with direct array indexing (no hashing needed)
// Memory: Most compact — no hash buckets, just an array

public enum Status { PENDING, IN_PROGRESS, COMPLETED, FAILED }

// ─── CREATION ──────────────────────────────────────────────────────────
Map<Status, Integer> statusCount = new EnumMap<>(Status.class);
statusCount.put(Status.PENDING, 10);
statusCount.put(Status.IN_PROGRESS, 5);
statusCount.put(Status.COMPLETED, 42);
statusCount.put(Status.FAILED, 3);

// ─── OPERATIONS ────────────────────────────────────────────────────────
statusCount.get(Status.PENDING);        // 10 — O(1) direct array[ordinal()]
statusCount.put(Status.IN_PROGRESS, 6); // O(1) update
statusCount.containsKey(Status.COMPLETED);  // true — O(1)
statusCount.remove(Status.FAILED);      // O(1)

// ─── ITERATION (always in ordinal order) ──────────────────────────────
for (Status status : statusCount.keySet()) {
    System.out.println(status + ": " + statusCount.get(status));
}
// Output (always in enum ordinal order):
// PENDING: 10
// IN_PROGRESS: 6
// COMPLETED: 42
// FAILED: key not found (or use getOrDefault)

// Use getOrDefault for missing keys
statusCount.getOrDefault(Status.FAILED, 0);  // 0 — default not in map

// ─── ENUM vs REGULAR MAP PERFORMANCE ───────────────────────────────────
// EnumMap: O(1) with NO hashing overhead
// HashMap: O(1) average but requires hash computation + bucket lookup

// In tight loops: EnumMap 10× faster (no hash computation, cache-friendly array)
```

### Map Implementation Comparison

| Aspect | `HashMap` | `LinkedHashMap` | `TreeMap` | `EnumMap` |
|--------|-----------|-----------------|-----------|-----------|
| **Iteration order** | Arbitrary | Insertion (or access) | Sorted by key | Ordinal (enum order) |
| **put/get/remove** | O(1) | O(1) | O(log n) | O(1) |
| **Iteration** | O(n + capacity) | O(n) | O(n) | O(n) |
| **Null keys** | One allowed | One allowed | Not allowed | Not allowed (enum) |
| **Performance** | General purpose | Preserve order, LRU | Range queries, sorted | Fastest for enum keys |
| **Use Case** | Default choice | Insertion order, caching | Sorted keys, ranges | Permission flags, status maps |

### Sorting Map Entries Dynamically

```java
// HashMap iteration order is arbitrary — to sort results:

Map<String, Integer> scores = Map.of("Charlie", 85, "Alice", 92, "Bob", 78);

// Sort by KEY ascending (natural order)
scores.entrySet().stream()
    .sorted(Map.Entry.comparingByKey())
    .forEach(e -> System.out.println(e.getKey() + "=" + e.getValue()));
// Alice=92, Bob=78, Charlie=85

// Sort by VALUE descending
scores.entrySet().stream()
    .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
    .forEach(e -> System.out.println(e.getKey() + "=" + e.getValue()));
// Alice=92, Charlie=85, Bob=78

// Collect sorted result into LinkedHashMap (preserves sorted iteration order)
Map<String, Integer> sortedByValue = scores.entrySet().stream()
    .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
    .collect(Collectors.toMap(
        Map.Entry::getKey,
        Map.Entry::getValue,
        (e1, e2) -> e1,        // merge function (no duplicates)
        LinkedHashMap::new     // CRUCIAL: LinkedHashMap preserves insertion order
    ));

// Now sortedByValue iterates in sorted order
sortedByValue.forEach((k, v) -> System.out.println(k + "=" + v));
// Alice=92, Charlie=85, Bob=78  (stays sorted)
```

### Map Implementations Comparison

| Aspect | `HashMap` | `LinkedHashMap` | `TreeMap` |
|--------|-----------|-----------------|-----------|
| **Iteration order** | Arbitrary | Insertion (or access) | Sorted by key |
| **put/get/remove** | O(1) | O(1) | O(log n) |
| **Iteration** | O(n + capacity) | O(n) | O(n) |
| **Null keys** | One allowed | One allowed | Not allowed (compareTo) |
| **Use Case** | General purpose | Preserve order, LRU | Range queries, sorted iteration |

### Interview Answer
> "HashMap for general-purpose fast key-value access — O(1) with no overhead. LinkedHashMap when you need insertion order (or access order for LRU cache via overriding `removeEldestEntry`). TreeMap when keys need to be sorted — O(log n) operations but range queries like `subMap` and `headMap` become natural.
>
> For LRU cache specifically, LinkedHashMap with access-order=true (third constructor parameter) is the standard approach — frequent accesses move items to the end, `removeEldestEntry` evicts the oldest (least recently used). In real systems I'd use Caffeine or Ehcache, but LinkedHashMap works for simple cases."
>
> *Likely follow-up: "How does LinkedHashMap maintain insertion order without sacrificing O(1) operations?"*

---

## Part 4: Sorting, Ordering & Comparable vs Comparator

## Q22 & Q23. Comparable vs Comparator — Natural vs Custom Ordering

### Concept

**Comparable** = "I know how to compare myself to other objects" (internal, one ordering)  
**Comparator** = "Here's an external way to compare two objects" (external, many orderings possible)

### Code — Comparable (Natural Ordering)

```java
// Natural ordering: Employee sorted by salary (ascending)
public class Employee implements Comparable<Employee> {
    private final String name;
    private final double salary;
    private final int age;

    public Employee(String name, double salary, int age) {
        this.name = name;
        this.salary = salary;
        this.age = age;
    }

    @Override
    public int compareTo(Employee other) {
        // Return: negative if this < other, 0 if equal, positive if this > other
        return Double.compare(this.salary, other.salary);  // Ascending salary order
    }

    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>(List.of(
            new Employee("Alice", 90000, 30),
            new Employee("Bob", 70000, 25),
            new Employee("Carol", 85000, 28)
        ));

        Collections.sort(employees);  // Uses compareTo()
        employees.forEach(e -> System.out.println(e.name + " $" + e.salary));
        // Bob $70000.0, Carol $85000.0, Alice $90000.0
    }
}

// Also works with TreeSet (maintains sorted order)
Set<Employee> empSet = new TreeSet<>(List.of(
    new Employee("Alice", 90000, 30),
    new Employee("Bob", 70000, 25),
    new Employee("Carol", 85000, 28)
));
// TreeSet automatically keeps sorted by compareTo()
```

### Code — Comparator (Custom Ordering, Multiple Strategies)

```java
// Now you can have MANY different sort orders without changing the class

List<Employee> employees = new ArrayList<>(List.of(
    new Employee("Alice", 90000, 30),
    new Employee("Bob", 70000, 25),
    new Employee("Carol", 85000, 28)
));

// Sort by name (alphabetical)
employees.sort(Comparator.comparing(Employee::getName));
// Alice $90000, Bob $70000, Carol $85000

// Sort by salary descending
employees.sort(Comparator.comparing(Employee::getSalary).reversed());
// Alice $90000, Carol $85000, Bob $70000

// Sort by salary ascending, then name for ties
employees.sort(
    Comparator.comparing(Employee::getSalary)
              .thenComparing(Employee::getName)
);

// Complex: by department descending, then by salary descending, then by age ascending
employees.sort(
    Comparator.comparing(Employee::getDepartment).reversed()
              .thenComparing(Employee::getSalary).reversed()
              .thenComparing(Employee::getAge)
);

// Null-safe comparator (nulls last)
employees.sort(
    Comparator.comparing(Employee::getAge, Comparator.nullsLast(Integer::compare))
);

// Custom comparator logic
Set<Employee> customeOrdered = new TreeSet<>((e1, e2) -> {
    if (!e1.getDepartment().equals(e2.getDepartment())) {
        return e1.getDepartment().compareTo(e2.getDepartment());  // By department
    }
    return Double.compare(e1.getSalary(), e2.getSalary());  // Then by salary
});
```

### Critical Contract: Consistency Between Comparable and equals

```java
// ⚠️ DANGEROUS: compareTo and equals disagree

public class BrokenEmployee implements Comparable<BrokenEmployee> {
    private String name;
    
    @Override
    public int compareTo(BrokenEmployee other) {
        return this.name.compareTo(other.name);  // Compare by name
    }
    
    @Override
    public boolean equals(Object o) {
        return this == o;  // Compare by identity ← INCONSISTENCY!
    }
}

// Now TreeSet behaves weirdly:
Set<BrokenEmployee> set = new TreeSet<>();
BrokenEmployee e1 = new BrokenEmployee("Alice");
BrokenEmployee e2 = new BrokenEmployee("Alice");

set.add(e1);
set.add(e2);

System.out.println(set.size());  // 2 !!! (e1 and e2 have compareTo==0 but !equals)
System.out.println(e1.equals(e2));  // false
System.out.println(set.contains(new BrokenEmployee("Alice")));  // false !!!

// TreeSet uses compareTo for its invariants, not equals
// If compareTo says "equal" but equals disagrees, chaos ensues
```

### Correct Pattern: Consistent Comparable and equals

```java
// ✓ CORRECT: compareTo and equals agree on what "equal" means

public class Employee implements Comparable<Employee> {
    private final String name;
    private final String id;
    
    @Override
    public int compareTo(Employee other) {
        return this.id.compareTo(other.id);  // Natural order: by ID
    }
    
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Employee e)) return false;
        return this.id.equals(e.id);  // Same meaning of "equal"
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}

// Now TreeSet behavior is predictable:
Set<Employee> set = new TreeSet<>();
set.add(new Employee("Alice", "E001"));
set.add(new Employee("Alice", "E001"));  // Duplicate → not added (both methods agree)
System.out.println(set.size());  // 1 ✓
```

### Interview Answer
> "`Comparable` defines the natural ordering of a class — what "less than" means for that type. One class, one natural order. You implement `compareTo` and objects are automatically sorted anywhere the framework expects it (TreeSet, TreeMap, Arrays.sort).
>
> `Comparator` defines external, custom orderings — you can have many for the same type. Use `Comparator` for one-off sortings, multiple sort criteria, or when you can't modify the class.
>
> Critical contract: if you implement `Comparable`, keep `compareTo` and `equals` consistent — if `compareTo` returns 0, `equals` should return true. Breaking this causes TreeSet/TreeMap to silently drop or behave unpredictably.
>
> I use `Comparable` for simple, stable natural orderings (ID, name). I use `Comparator.comparing()` chains for multi-level sorts and one-off operations."
>
> *Likely follow-up: "What happens if compareTo is inconsistent with equals in a TreeSet?"*

---

## Part 5: Generics in Collections — Type Safety & Usage

## Generics Fundamentals with Collections

### Concept
Generics enable **compile-time type safety** for collections, preventing `ClassCastException` at runtime and eliminating explicit casts.

### Code — Raw Types vs Generics

```java
// ❌ RAW TYPE (Java 1.4 style) — NO type safety
List names = new ArrayList();    // No generic parameter
names.add("Alice");
names.add(42);                   // Compiler doesn't complain!
names.add(new User());           // Any type allowed
String name = (String) names.get(0);     // Manual cast required ← ClassCastException risk!
// names.get(1) is actually Integer, not String → Type mismatch!

// ✅ WITH GENERICS — Compile-time safety
List<String> names = new ArrayList<>();
names.add("Alice");              // ✓ Compiler approves
// names.add(42);                // ✗ COMPILE ERROR — type mismatch
// names.add(new User());        // ✗ COMPILE ERROR
String name = names.get(0);      // No cast needed, already String ✓

// No unchecked warnings, no runtime surprises
```

### Type Erasure — How Generics Work Under the Hood

```java
// Generics are ERASED at runtime — JVM only sees raw types

List<String> stringList = new ArrayList<>();
List<Integer> intList = new ArrayList<>();

// Same runtime class!
System.out.println(stringList.getClass() == intList.getClass());  // true
// Both compiled to ArrayList without type info

// This is why you can't do:
public <T> void printList(List<T> list) {
    // if (list instanceof List<String>) { }  ← COMPILE ERROR
    if (list instanceof List) { }             // ✓ Only raw type at runtime
}

// Type erasure consequences:
List<?> list = new ArrayList<String>();
// list.get(0);  ← Unknown element type, even though it's a List<String>

// Implications for arrays vs generics:
T[] array = new T[10];  // ← COMPILE ERROR (arrays need reifiable type info)
List<T> list = new ArrayList<>();  // ✓ Generics work with erasure
```

### Bounded Type Parameters — Constraining Generics

```java
// ─── UPPER BOUNDED: T extends SomeClass ────────────────────────────────
// T is any type that IS-A SomeClass

interface Repository<T extends User> {
    void save(T user);
    T findById(Long id);
}

class AdminRepository implements Repository<Admin> {
    @Override
    public void save(Admin admin) { /* ... */ }
    @Override
    public Admin findById(Long id) { /* ... */ }
}

// Only Admin, Manager, or other User subclasses allowed
// Admin IS-A User, so it works

// ─── MULTIPLE BOUNDS ──────────────────────────────────────────────────
// T extends Class1 & Interface1 & Interface2
public <T extends Comparable<T> & Serializable> void sort(List<T> list) {
    // T must be both Comparable and Serializable
    Collections.sort(list);
}

// ─── LOWER BOUNDED: ? super Type ──────────────────────────────────────
// Wildcard that accepts Type or any superclass
List<? super Manager> list = new ArrayList<User>();  // User is superclass of Manager
list.add(new Manager());  // OK — Manager IS-A User
// list.get();  ← Returns Object, not Manager (covariance works down, not up)
```

### Variance: Covariance, Contravariance, Invariance

```java
// INVARIANT: List<Manager> is NOT a List<User>, even though Manager IS-A User
List<Manager> managers = new ArrayList<>();
// List<User> users = managers;  // ← COMPILE ERROR
// Prevents: users.add(new User());  which would corrupt the managers list

// —— COVARIANT: ? extends Type ——————————————————————————————
// Producer: read-only, upper bound
public List<? extends User> getUsers() {
    return new ArrayList<Manager>();  // ✓ Manager IS-A User
}

List<? extends User> users = getUsers();
User user = users.get(0);  // OK, returns User
// users.add(new Manager());  // ← COMPILE ERROR (can't write with upper bound)

// —— CONTRAVARIANT: ? super Type ———————————————————————————
// Consumer: write-only, lower bound
public void saveUsers(List<? super Manager> managers) {
    managers.add(new Manager());  // ✓ Manager IS-A whatever type
    // Manager m = managers.get(0);  // ← COMPILE ERROR (returns Object, not Manager)
}

saveUsers(new ArrayList<User>());  // ✓ User is superclass of Manager
```

### PECS Rule: Producer-Extends, Consumer-Super

```java
// When designing generic methods, remember PECS:
// — Producer (reading from collection): use extends (covariance)
// — Consumer (writing to collection): use super (contravariance)

// PRODUCER: copy FROM source to destination (reading from source)
public static <T> void copy(List<? extends T> source, List<? super T> dest) {
    for (T item : source) {  // source produces items of type T
        dest.add(item);       // destination consumes items of type T
    }
}

List<Manager> managers = List.of(new Manager(), new Manager());
List<User> users = new ArrayList<>();
copy(managers, users);  // ✓ Correct — source produces Manager (extends User), dest consumes T
// Manager extends User → fits ? extends T
// User extends User → fits ? super User

// ——— Real-world example: Stream API follows PECS ———————————
public interface Stream<T> {
    <R> Stream<R> map(Function<? super T, ? extends R> mapper);
    // Mapper consumes T (? super T) and produces R (? extends R)
    
    void forEach(Consumer<? super T> action);
    // Consumer consumes T (? super T)
    
    <R> R collect(Collector<? super T, ?, R> collector);
    // Collector consumes T
}
```

### Generics in Collections — Practical Patterns

```java
// ━━━ Building type-safe collections ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

// Generic wrapper around HashMap
public class Repository<K, V> {
    private final Map<K, V> data = new HashMap<>();
    
    public void add(K key, V value) {
        data.put(key, value);
    }
    
    public V get(K key) {
        return data.get(key);
    }
}

Repository<String, User> userRepo = new Repository<>();
userRepo.add("user:1", new User("Alice"));
User alice = userRepo.get("user:1");  // Already typed User, no cast

// ━━━ Generic utility methods ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

public class CollectionUtils {
    // Find first element matching predicate
    public static <T> Optional<T> findFirst(Collection<T> c, Predicate<? super T> predicate) {
        return c.stream().filter(predicate).findFirst();
    }
    
    // Transform collection using mapper
    public static <T, R> List<R> map(List<? extends T> source, Function<? super T, ? extends R> mapper) {
        return source.stream().map(mapper).toList();
    }
    
    // Flatten nested collections
    public static <T> List<T> flatten(List<? extends List<? extends T>> nested) {
        return nested.stream().flatMap(List::stream).toList();
    }
}

// Usage:
List<User> users = List.of(/* ... */);
Optional<User> admin = CollectionUtils.findFirst(users, u -> u.isAdmin());
List<String> names = CollectionUtils.map(users, User::getName);
```

### Interview Answer
> "Generics enable compile-time type safety — the compiler catches type mismatches that would be runtime exceptions (ClassCastException) without them. Generic types are erased at runtime — the JVM only sees raw types, which is why you can't do `new T[]` or `instanceof List<String>`.
>
> For collection signatures: use covariance (`? extends T`) when reading/producing (producers), contravariance (`? super T`) when writing/consuming. The acronym is PECS.
>
> In practice: always use generics, never raw types (unless dealing with legacy code). Use bounded types (`<T extends Foo>`) when the type must have certain capabilities. For real-world collections, prefer immutable return types like `List<E>` instead of `ArrayList<E>` — it's more flexible and follows the Interface Segregation Principle."
>
> *Likely follow-up: "What is type erasure and what are its consequences?"*

---

## Q24. HashSet Internals — Uses HashMap Under the Hood

### Concept
`HashSet<E>` is literally a `HashMap<E, Object>` where all values are the same dummy `Object` constant. You're storing elements as keys.

### How It Works
```java
// From JDK source (simplified):
public class HashSet<E> {
    private static final Object PRESENT = new Object();  // Dummy value for all entries
    private transient HashMap<E, Object> map;

    public boolean add(E e) {
        return map.put(e, PRESENT) == null;  // put returns null if key was new → true
        // returns previous value (PRESENT) if key existed → false (no duplicate added)
    }

    public boolean contains(Object o) {
        return map.containsKey(o);           // O(1) hash lookup
    }

    public boolean remove(Object o) {
        return map.remove(o) == PRESENT;
    }
}
```

```java
// Demonstration:
Set<String> names = new HashSet<>();
names.add("Alice");   // map.put("Alice", PRESENT) → null → true (added)
names.add("Bob");
names.add("Alice");   // map.put("Alice", PRESENT) → PRESENT → false (not added again)
System.out.println(names.size());  // 2

// Why does HashSet need equals + hashCode?
// Same reason HashMap does — keys are compared by hash then equals
// If you store custom objects, override BOTH methods:
public class Product {
    private String sku;

    @Override
    public int hashCode() { return Objects.hash(sku); }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Product p)) return false;
        return Objects.equals(sku, p.sku);
    }
}

Set<Product> products = new HashSet<>();
products.add(new Product("SKU-001"));
products.add(new Product("SKU-001"));  // Same sku → same hashCode → equals returns true → not added
System.out.println(products.size());   // 1 ✓
```

### Interview Answer
> "HashSet is fundamentally a HashMap where all values are a dummy Object constant — the elements are stored as keys. When you add an element, it calls `put(element, PRESENT)` and returns null if the key was new (add succeeds), or the existing value (PRESENT) if the key existed (add fails). This means HashSet inherits HashMap's O(1) average performance but also its requirements: you must properly override `hashCode` and `equals` on your custom objects. Two Employee objects with identical names must have identical hashCode and return true from equals, or they'll be treated as different keys and both will be stored, defeating the uniqueness guarantee."
>
> *Likely follow-up: "Why does HashSet need equals AND hashCode, not just one?"*

---

## Q25. Heterogeneous List — Mixed Types

```java
// Option 1: Raw List (no generics) — NOT recommended, compiler warnings
List mixed = new ArrayList();
mixed.add("string");
mixed.add(42);
mixed.add(new User());
String s = (String) mixed.get(0);  // Manual cast — ClassCastException risk

// Option 2: List<Object> — explicit but loses type safety
List<Object> list = new ArrayList<>();
list.add("Alice");
list.add(100);
list.add(LocalDate.now());

// Option 3: Sealed Interface (Java 17) — type-safe heterogeneous list
sealed interface Payload permits TextPayload, NumberPayload, DatePayload {}
record TextPayload(String value)      implements Payload {}
record NumberPayload(int value)       implements Payload {}
record DatePayload(LocalDate value)   implements Payload {}

List<Payload> payloads = List.of(
    new TextPayload("hello"),
    new NumberPayload(42),
    new DatePayload(LocalDate.now())
);

// Exhaustive pattern matching — no ClassCastException possible
for (Payload p : payloads) {
    String result = switch (p) {
        case TextPayload   t -> "Text: "   + t.value();
        case NumberPayload n -> "Number: " + n.value();
        case DatePayload   d -> "Date: "   + d.value();
    };
    System.out.println(result);
}
```

**Trade-offs:**
- `List<Object>`: flexible, but every access requires instanceof or cast
- Sealed + records: verbose setup, but **compiler-enforced exhaustive handling** — the best type-safe approach

### Interview Answer
> "The problem with heterogeneous lists (mixed types) is losing compile-time type safety. A raw `List` gives compiler warnings and ClassCastException risks at runtime. `List<Object>` is explicit but requires casting on every access. The modern Java 17+ solution is sealed interfaces with pattern matching — you define a sealed interface, create records for each type, then use exhaustive switch expressions. The compiler forces you to handle every case, eliminating the ClassCastException risk entirely. Sealed classes with records are verbose upfront but provide compile-time exhaustiveness checking, which beats both raw types and defensive casting."
>
> *Likely follow-up: "What is type elegance and when is a sealed interface a better choice than inheritance?"*

---

## Part 6: HashMap Internals Deep-Dive

## Q26. HashMap Architecture — Buckets, Hash Function, Collisions

### Concept

`HashMap<K,V>` stores entries in an **array of buckets**. `hashCode()` determines which bucket. Multiple entries in the same bucket are **chained** (linked list or tree since Java 8).

### Simple Explanation

Imagine 16 pigeonholes on a wall (the backing array). To store a key-value pair, the key is hashed to determine which pigeonhole. If multiple keys hash to the same pigeonhole, they're chained inside that hole.

### Internal Architecture Detailed

```
Default HashMap backing array (capacity = 16, load factor = 0.75, threshold = 12):

┌──────────────────────────────────────────────────────────────────────────┐
│ Index 0:  null                                                           │
│ Index 1:  Node(hash=65578, key="Alice", val=30) → null                    │
│ Index 3:  Node(hash=3122, key="Bob", val=25) → Node(hash=3122, key="Ca... │ ← Collision
│ Index 5:  null                                                           │
│ ...                                                                      │
│ Index 15: null                                                           │
└──────────────────────────────────────────────────────────────────────────┘

Node structure (JDK source):
public static class Node<K,V> {
    final int hash;       // Cached hash code of key
    final K key;
    V value;
    Node<K,V> next;       // Points to next node in the chain (collision handling)
}
```

### put(key, value) Flow Step-by-Step

```java
// Example: map.put("Alice", 30)

// Step 1: Compute hash of key
int hash = hash("Alice");              // hash("Alice") returns some integer
// JDK applies bit spreading for better distribution:
hash = hash ^ (hash >>> 16);          // Reduces collisions in low-order bits

// Step 2: Compute bucket index
int index = hash & (capacity - 1);    // capacity=16: hash & 15
// & operation (bitwise AND with 15) gives index in [0..15]

// Step 3: Access bucket
Node<K, V> bucket = table[index];

// Step 4: Check bucket contents
if (bucket == null) {
    // Empty bucket — create new node
    table[index] = new Node(hash, "Alice", 30, null);
    size++;
} else {
    // Collision — walk the chain (linked list or tree in Java 8+)
    Node<K, V> current = bucket;
    while (current != null) {
        if (current.hash == hash && (current.key == "Alice" || "Alice".equals(current.key))) {
            // KEY FOUND — update value
            current.value = 30;
            return;  // Don't increase size
        }
        if (current.next == null) {
            // Reached end of chain — key not found, append new node
            current.next = new Node(hash, "Alice", 30, null);
            size++;
            // Check if chain is too long — treeify if ≥ 8 (Java 8)
            break;
        }
        current = current.next;
    }
}

// Step 5: Check load factor
if (size > threshold) {  // threshold = capacity × 0.75
    resize();  // Double capacity, rehash all entries
}
```

### get(key) Flow Step-by-Step

```java
// Example: map.get("Alice")

// Step 1: Compute hash
int hash = hash("Alice");
hash = hash ^ (hash >>> 16);

// Step 2: Locate bucket
int index = hash & (capacity - 1);
Node<K, V> bucket = table[index];

// Step 3: Walk the chain
Node<K, V> current = bucket;
while (current != null) {
    // Check hash first (cheap int comparison)
    if (current.hash == hash) {
        // Hash matches, now check equality
        if (current.key == "Alice" || "Alice".equals(current.key)) {
            return current.value;  // FOUND
        }
    }
    // Hash mismatch — skip (don't even call equals for unrelated)
    current = current.next;
}

// Reached end of chain without match
return null;  // NOT FOUND
```

### Critical Memory Layout & Performance

```java
// The hash MUST be cached in the Node to avoid recomputation:
if (current.hash == hash) { ... }  // ✓ O(1) int comparison
// Without cached hash: might need to recompute for each collision check

// Hash check first, equals second (short-circuit on hash mismatch):
if (hash == h && (k == key || (key != null && key.equals(k))))
// Saves expensive equals() calls on unrelated collisions
```

### Resize and Rehashing

```java
// When size > capacity * 0.75, double the capacity and rehash

void resize() {
    int newCapacity = capacity * 2;        // 16 → 32 → 64 → ...
    Node[] newTable = new Node[newCapacity];
    
    // Rehash every existing entry
    for (Node<K, V> entry : oldTable) {
        if (entry != null) {
            Node<K, V> current = entry;
            while (current != null) {
                // Recompute index with new capacity
                int newIndex = current.hash & (newCapacity - 1);
                Node<K, V> next = current.next;
                current.next = newTable[newIndex];
                newTable[newIndex] = current;
                current = next;
            }
        }
    }
    
    table = newTable;
    capacity = newCapacity;
    threshold = (int)(capacity * 0.75f);
}

// Complexity: O(n) where n = size(), but amortised to O(1) per insert
// Why? Doubling capacity → each entry is rehashed at most log(final capacity) times
```

### Interview Answer
> "HashMap uses an array of buckets. `hashCode()` is hashed to produce an integer, then masked to the array index via `hash & (capacity-1)`. Multiple keys landing on the same index is a collision — they chain as a linked list (or Red-Black tree in Java 8+ if chain ≥ 8 nodes).
>
> `get()` jumps to the bucket, then walks the chain comparing hash first (fast int comparison), then equals (potentially expensive). If hashes differ, we skip the equals call — this is a critical optimization.
>
> When size exceeds `capacity × loadFactor` (default 0.75), the array doubles in size and every entry is rehashed — O(n) but amortised to O(1) per insert over time.
>
> Why 0.75 load factor? It's a tradeoff: lower = fewer collisions but more memory wasted; higher = better memory use but more collisions. 0.75 is empirically optimal."
>
> *Likely follow-up: "What happens to an Entry when HashMap resizes?"*

---

## Q27. hashCode Contract and Design

### Concept

`hashCode()` maps an object to a bucket in O(1). Without it, finding a key requires O(n) full scan.

### The Contract

```
MANDATORY:
  If a.equals(b) → true,  then a.hashCode() MUST == b.hashCode()

ALLOWED (not a contract violation):
  If a.hashCode() == b.hashCode() → a.equals(b) might be FALSE (collision, ok)
  If a.equals(b) → false, hashCode() can be same or different

BREAKING THE CONTRACT:
equals() says "equal" but hashCode() says "different bucket" → KEY LOST in map
```

### Code — The Bug

```java
// ❌ Problem: no hashCode override (uses Object default = memory address)
class Employee {
    String name;
    Employee(String name) { this.name = name; }
}

Map<Employee, String> map = new HashMap<>();
Employee e1 = new Employee("Alice");
map.put(e1, "Engineer");

Employee e2 = new Employee("Alice");  // Same data, different object in memory
System.out.println(map.get(e2));      // null! e2 is a "different key"

// Why?
// e1.hashCode() = address of e1 object (say, 42)
// e2.hashCode() = address of e2 object (say, 87) ← different!
// map.get(e2) looks in bucket 87, but e1 is in bucket 42
// Never found, returns null
```

### Correct Implementation

```java
// ✓ Correct: Override hashCode based on identity fields

public class Employee {
    private final String name;

    public Employee(String name) {
        this.name = Objects.requireNonNull(name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name);  // ← Same name = same hash = same bucket
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee emp)) return false;
        return Objects.equals(name, emp.name);  // ← Same name = equal
    }
}

// Now:
Map<Employee, String> map = new HashMap<>();
Employee e1 = new Employee("Alice");
map.put(e1, "Engineer");

Employee e2 = new Employee("Alice");
System.out.println(map.get(e2));            // "Engineer" ✓
System.out.println(e1.equals(e2));          // true ✓
System.out.println(e1.hashCode() == e2.hashCode());  // true ✓
```

### The 31 Prime Technique (Understanding Legacy Code)

```java
// Manual hashCode implementation (what to understand, though Objects.hash is cleaner)

public class Employee {
    private String name;
    private int age;

    @Override
    public int hashCode() {
        int result = 17;                    // Non-zero prime start
        result = 31 * result + (name != null ? name.hashCode() : 0);
        result = 31 * result + age;
        return result;
    }
}

// Why 31? 
// — It's prime (reduces collisions)
// — 31*n = (n << 5) - n (shift left 5, minus n) — JVM optimizes this to a single operation
// — Not too large, fits in 32 bits

// Magic: 31 * h = 32*h - h = (h << 5) - h
int hash = field.hashCode();
int result = 31 * result + hash;          // ~1 CPU cycle with JIT optimization
// vs a full multiplication for other numbers — this matters in tight loops
```

### Consequences of Breaking the Contract

```java
// ❌ Hash changes after object is in the map

public class MutableEmployee {
    private String name;
    
    public void setName(String name) { this.name = name; }  // MUTATION after storing!
    
    @Override
    public int hashCode() { return Objects.hash(name); }
}

Map<MutableEmployee, String> map = new HashMap<>();
MutableEmployee emp = new MutableEmployee("Alice");
emp.setName("Alice");
map.put(emp, "Engineer");

emp.setName("Bob");                        // Change name AFTER inserting
System.out.println(map.get(new MutableEmployee("Bob")));  // null!

// Why? emp's hashCode changed, so a new query goes to different bucket
// Original emp is in wrong bucket (where "Alice" is), unreachable

// KEY RULE: Keys in HashMap MUST BE EFFECTIVELY IMMUTABLE
// Mutate them and the map becomes corrupted
```

### Interview Answer
> "Without `hashCode()`, HashMap would scan every entry — O(n) always. `hashCode()` gives the bucket in O(1). The contract is ironclad: equal objects MUST have equal hashes. I use `Objects.hash(field1, field2, ...)` — idiomatic and handles nulls.
>
> The common failure: forgetting to override `hashCode` when you override `equals`. Equal objects end up in different buckets and never find each other.
>
> Another critical rule: keys must be effectively immutable. If you change a key's fields after inserting it, the map becomes corrupted — it's in the wrong bucket and becomes unreachable."
>
> *Likely follow-up: "What is the hashCode contract?"*

---

## Q28. Collisions and Java 8 Treeification

### Concept

Collisions occur when two different keys hash to the same bucket index. Java 8 fixes the worst-case O(n) chain lookup by converting long chains to Red-Black trees.

### Why Collisions Happen

```java
// Infinite keys, finite buckets (pigeonhole principle)
// With capacity 16, at most 16 unique hash indices [0..15]
// Eventually, multiple keys land in the same index

// Example:
hash("Alice") & 15 = 3
hash("Carol") & 15 = 3   // Different keys, same bucket!

// Degenerate catastrophe: constant hashCode
class BadKey {
    int value;
    @Override
    public int hashCode() { return 42; }  // ALWAYS returns 42!
}

Map<BadKey, String> map = new HashMap<>();
for (int i = 0; i < 100; i++) {
    map.put(new BadKey(), "value" + i);  // All go to same bucket!
}

// HashMap degenerates to linked list, all operations become O(n)
map.get(new BadKey());  // Walks all 100 entries in one chain ← catastrophic!
```

### Java 8 Treeification

```
BEFORE Java 8:
  Bucket with collision → LinkedList chain, O(n) worst case in one bucket

JAVA 8+:
  Chain length ≤ 7   → Keep as LinkedList, O(n)
  Chain length = 8   → TREEIFY: convert to Red-Black Tree, O(log n)
  Tree size shrinks to 6 → UNTREEIFY: convert back to LinkedList

JDK constants (source: HashMap.java):
  static final int TREEIFY_THRESHOLD = 8;
  static final int UNTREEIFY_THRESHOLD = 6;
  static final int MIN_TREEIFY_CAPACITY = 64;  // Don't treeify with small table
```

### Code — Before and After

```java
// BEFORE Java 8 with bad hash:
// 100 collisions → one chain of 100 nodes
Node chain:  Node → Node → Node → ... → Node(100th)
lookup:      Start at head, walk 50 on average ← O(50) = O(n)

// AFTER Java 8 with bad hash:
// 100 collisions → automatic conversion to tree
TreeNode<K,V> (Red-Black tree)
height ≈ log(100) ≈ 7
lookup:      Binary tree search ← O(7) = O(log n)

// Benefit: catastrophic hash functions no longer destroy performance
```

### Why Threshold 8?

```
Statistical analysis of collision frequency:
  With a good hash function and load factor 0.75:
  Probability of 8 collisions in one bucket ≈ 0.00000002 (from Poisson distribution)
  
  This means:
  — 8 is rare enough that tree conversion overhead is justified
  — Tree overhead (more memory, pointer chasing) is paid infrequently
  — Reduces pathological case (all entries hash to same bucket)
```

### Interview Answer
> "Collisions happen because there are infinite keys but limited buckets. Before Java 8, a long chain of collisions was O(n) lookup. Java 8 fixes this: once a bucket's chain reaches 8 entries, it's converted to a Red-Black tree, reducing worst-case to O(log n).
>
> The threshold of 8 is based on statistics: with a good hash function and standard load factor, having 8 collisions in one bucket is so rare (~1 in 50 million maps) that paying the tree overhead is worthwhile for the worst-case guarantee.
>
> The table must have at least 64 buckets before treeifying — with fewer buckets, just resizing is better than treeifying."
>
> *Likely follow-up: "What is a Red-Black tree and why is it used?"*

---

## Part 7: Thread-Safe Collections

## ConcurrentHashMap — Multi-threaded Hashing

### Concept

`ConcurrentHashMap` allows concurrent reads and fine-grained write locking without locking the entire map. Multiple threads can write to different buckets simultaneously.

### Architecture

```
TRADITIONAL SYNCHRONOUS APPROACHES:
  Collections.synchronizedMap(new HashMap()) → Global lock on all operations
  Hashtable → Global lock (synchronized all methods)
  Result: Only one thread can access map at a time → poor throughput

JAVA 5-7: SEGMENTED LOCKING
  16 segments, each with its own lock
  Thread 1 locks segment 0, Thread 2 locks segment 8 simultaneously → 2× throughput
  But 16 threads competing for 16 segments = contention

JAVA 8+: BUCKET-LEVEL LOCKING
  — Reads: LOCK-FREE using volatile fields (no synchronization)
  — Writes: CAS (Compare-And-Swap) for uncontended cases
  — Writes: synchronized on individual bucket HEAD for contested cases
  Result: Multiple threads writing to different buckets → no contention
```

### Code

```java
// ❌ NOT thread-safe
Map<String, Integer> unsafeMap = new HashMap<>();

// ✅ Thread-safe but slow: global lock
Map<String, Integer> slowMap = Collections.synchronizedMap(new HashMap<>());

// ✅ CORRECT: fine-grained locking
Map<String, Integer> concMap = new ConcurrentHashMap<>();

// Critical: atomic operations (normal check-then-act is not atomic)
// ❌ WRONG: check-then-put is not atomic under concurrency
if (!concMap.containsKey("key")) {
    concMap.put("key", 1);
}
// Race: Thread 1 checks (key absent) and Thread 2 checks (key absent) in parallel
// Both put → both think they successfully inserted → one overwrites the other

// ✅ CORRECT: Use atomic methods
concMap.putIfAbsent("key", 1);                    // Insert only if absent, atomic
concMap.computeIfAbsent("key", k -> expensive()); // Compute and insert if absent
concMap.compute("count", (k, v) -> v == null ? 1 : v + 1);  // Atomic update
concMap.merge("count", 1, Integer::sum);           // Atomic merge

// Building frequency map atomically
Map<String, Integer> freq = new ConcurrentHashMap<>();
words.stream().parallel().forEach(word -> 
    freq.merge(word, 1, (old, one) -> old + one)  // Atomic increment
);
```

### ConcurrentHashMap vs Alternatives

| | `ConcurrentHashMap` | `Collections.synchronized` | `Hashtable` |
|---|---|---|---|
| **Lock scope** | Single bucket | Entire map | Entire map |
| **Read throughput** | Very high (no lock) | Low (global lock) | Low |
| **Write contention** | Low (multiple buckets) | High (one lock) | High |
| **Null keys/values** | No (throws NPE) | Yes | Yes |
| **Iterators** | Weakly consistent | Fail-fast | Fail-fast |
| **Recommendation** | ✓ Production choice | Rarely | Never |

```java
// ConcurrentHashMap disallows nulls — why?
concMap.put(null, "value");  // NullPointerException!
concMap.put("key", null);    // NullPointerException!

// Reason: Ambiguity
concMap.get("key");  // Returns null
// Is the key absent? Or is the value null?
// In single-threaded HashMap you can check containsKey, but in concurrent context
// between get() and containsKey() another thread might change the value
// So nulls are forbidden to avoid this ambiguity
```

### putIfAbsent vs computeIfAbsent — Critical Difference

```java
// putIfAbsent: EAGER — value is COMPUTED BEFORE calling
String key = "expensive_key";
String value = expensiveCalculation();  // ← Always runs, even if key exists!
concMap.putIfAbsent(key, value);        // Then insert if absent
// Waste: if key already exists, we wasted CPU computing the unused value

// computeIfAbsent: LAZY — function is ONLY called if key is absent
concMap.computeIfAbsent(key, k -> expensiveCalculation());
// Function only runs if key is absent — efficient!

// Real example: caching expensive computation
ConcurrentHashMap<String, User> userCache = new ConcurrentHashMap<>();

// ❌ INEFFICIENT: Loads user even if already cached
User user = userService.loadUser(userId);  // DB call
userCache.putIfAbsent(userId, user);       // Then check cache

// ✅ EFFICIENT: Only loads if not cached
User user = userCache.computeIfAbsent(userId, id -> userService.loadUser(id));
```

### ConcurrentHashMap Iterators — Weakly Consistent

```java
// HashMap iterator is FAIL-FAST: throws ConcurrentModificationException if modified during iteration
Map<String, Integer> map = new HashMap<>();
for (String key : map.keySet()) {
    if (key.equals("toRemove")) {
        map.remove(key);  // ← ConcurrentModificationException!
    }
}

// ConcurrentHashMap iterator is WEAKLY CONSISTENT
Map<String, Integer> concMap = new ConcurrentHashMap<>();
for (String key : concMap.keySet()) {
    if (key.equals("toRemove")) {
        concMap.remove(key);  // ← NO EXCEPTION! Iterator reflects state from creation
    }
}
// Iterator may or may not reflect the removal (depends on timing)
// But NEVER throws ConcurrentModificationException
// This allows concurrent modification without breaking iteration
```

### CopyOnWriteArrayList — Thread-Safe List Alternative

```java
// For lists needing thread safety, use CopyOnWriteArrayList (not ConcurrentHashMap)
// Strategy: Copy entire array on every write (expensive), but reads are lock-free

List<String> readHeavy = new CopyOnWriteArrayList<>();
readHeavy.add("a");
readHeavy.add("b");  // Copies entire array [a] → [a,b]

// Perfect for: read-heavy (1000 reads, 10 writes)
// Terrible for: write-heavy (frequent modifications)

// Production note: Unlike ConcurrentHashMap, there's no CopyOnWriteHashMap
// For concurrent maps, ConcurrentHashMap is the only standard option
```

### Interview Answer
> "ConcurrentHashMap is the production choice for multi-threaded key-value storage. Unlike Hashtable or synchronized wrappers which lock the entire map, ConcurrentHashMap uses bucket-level locking — only the specific bucket being written is locked, so multiple threads can write to different buckets simultaneously without contention.
>
> Critical pattern: Use `computeIfAbsent(key, function)` — NOT `putIfAbsent(key, value)` — because computeIfAbsent is lazy (only computes if key absent). putIfAbsent always evaluates, wasting computation if key exists.
>
> Unlike HashMap, null keys and values are forbidden to avoid ambiguity: get returning null could mean 'key absent' or 'value is null', which is dangerous in concurrent scenarios.
>
> Iterators are 'weakly consistent' — reflect the map state from iterator creation but never throw ConcurrentModificationException, allowing safe concurrent reads during modifications."
>
> *Likely follow-up: "Why would you use CopyOnWriteArrayList when ConcurrentHashMap exists for maps?"*

---

## Part 7: Fail-Fast vs Fail-Safe Iterators

### Concept

**Fail-Fast**: Iterator throws `ConcurrentModificationException` if collection is modified during iteration (safety over permissiveness)  
**Fail-Safe**: Iterator works on a copy or snapshot; modifications don't affect iteration (permissiveness over safety)

### Code — Fail-Fast (HashMap, ArrayList, HashSet)

```java
// FAIL-FAST iterators: most common collections use this strategy
List<String> list = new ArrayList<>(List.of("a", "b", "c"));

// ❌ This throws ConcurrentModificationException
Iterator<String> iter = list.iterator();
while (iter.hasNext()) {
    String item = iter.next();
    if (item.equals("b")) {
        list.remove(item);  // ← Modifying collection during iteration!
    }
}
// throws: java.util.ConcurrentModificationException

// ✅ CORRECT: Use iterator.remove()
Iterator<String> iter = list.iterator();
while (iter.hasNext()) {
    String item = iter.next();
    if (item.equals("b")) {
        iter.remove();  // ← Remove via iterator, safe!
    }
}

// ✅ ALTERNATIVE: Use removeIf (Java 8+)
list.removeIf(item -> item.equals("b"));

// ✅ ALTERNATIVE: Iteration on a copy
List<String> copy = new ArrayList<>(list);
for (String item : copy) {
    list.remove(item);  // Remove from original, iterate over copy
}

// Implementation: ArrayList maintains modCount (internal modification counter)
// Iterator checks modCount at start and compares on each next()
// If modCount changed, throws ConcurrentModificationException
```

### Code — Fail-Safe (ConcurrentHashMap, CopyOnWriteArrayList)

```java
// FAIL-SAFE iterators: concurrent collections use this strategy
List<String> list = new CopyOnWriteArrayList<>(List.of("a", "b", "c"));

// ✅ NO EXCEPTION (Iterator is fail-safe — works on snapshot)
Iterator<String> iter = list.iterator();
while (iter.hasNext()) {
    String item = iter.next();
    if (item.equals("b")) {
        list.remove(item);  // Modifying original list, OK!
    }
}
// No exception! Modification doesn't affect iterator

// Similarly with ConcurrentHashMap
Map<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1);
map.put("b", 2);

for (String key : map.keySet()) {
    if (key.equals("b")) {
        map.remove(key);  // ← OK with ConcurrentHashMap!
    }
}
// No exception, but iterator may or may not reflect the removal
```

### Trade-offs

| Aspect | Fail-Fast | Fail-Safe |
|--------|-----------|-----------|
| **Implementation** | Works on actual collection | Works on copy/snapshot |
| **Memory** | Minimal overhead | Overhead of copy |
| **Performance** | Fast iteration | Slower (copy required) |
| **Concurrent mods** | Detected (exception) | Allowed/ignored |
| **Use Case** | General collections | Concurrent/multi-threaded |
| **Examples** | ArrayList, HashMap, HashSet | CopyOnWriteArrayList, ConcurrentHashMap |

### Performance Impact

```java
// Fail-safe is more expensive for modifications
List<String> list = new CopyOnWriteArrayList<>();

// Each add() copies the entire array!
for (int i = 0; i < 1000; i++) {
    list.add("item");  // O(n) copy per add!
}
// Total: O(n²) for write-heavy workload — terrible!

// ArrayList is much faster for writes
List<String> list2 = new ArrayList<>();
for (int i = 0; i < 1000; i++) {
    list2.add("item");  // O(1) amortised append
}
// Total: O(n) for write-heavy workload

// CopyOnWriteArrayList is perfect for READ-HEAVY workloads
// (1000 reads, 10 writes) — reads are lock-free, writes copy array
```

### Interview Answer
> "ArrayList, HashMap, HashSet use fail-fast iterators — they throw `ConcurrentModificationException` if the collection is modified during iteration. This is a safety mechanism. To modify during iteration, use `iterator.remove()` or `removeIf()`, or iterate over a copy.
>
> ConcurrentHashMap and CopyOnWriteArrayList use fail-safe iterators — they work on snapshots/copies, so modifications are safe but iterations may not reflect recent changes. CopyOnWriteArrayList copies the entire array on every write, so it's only suitable for read-heavy scenarios.
>
> In single-threaded code, just avoid modifying during iteration. In concurrent code, use atomic methods on ConcurrentHashMap or embrace the fail-safe semantics of CopyOnWriteArrayList for appropriate workloads."
>
> *Likely follow-up: "When would you use CopyOnWriteArrayList over ArrayList?"*

---

## Part 8: Utility Classes & Advanced Patterns

## Collections, Arrays, and Utility Methods

```java
// ─── Collections utility methods ────────────────────────────────────────

// Immutable collections (defensive copies)
List<String> immutable = Collections.unmodifiableList(new ArrayList<>(myList));
immutable.add("new");  // throws UnsupportedOperationException

// Sorting with null-safe comparator
List<Integer> nums = new ArrayList<>(List.of(1, null, 3));
Collections.sort(nums, Comparator.nullsFirst(Integer::compareTo));  // [null, 1, 3]

// Frequency counting
Map<String, Integer> freq = Collections.frequency("hello", 'l');  // Oops, wrong API
// Use stream instead:
Map<String, Integer> freq = words.stream()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));

// ─── Map utility methods (Java 8+) ──────────────────────────────────────

Map<String, User> users = new HashMap<>();

// Get with default
User alice = users.getOrDefault("alice", User.GUEST);

// Compute if absent (lazy evaluation — function only called if absent)
users.computeIfAbsent("bob", id -> loadUserFromDB(id));

// Compute value (called always)
users.compute("charlie", (id, existing) -> refreshFromDB(id, existing));

// Merge (combine values if key exists)
users.merge("alice", newUser, (old, new) -> old.merge(newUser));

// ─── Streams with collections ───────────────────────────────────────────

List<User> users = new ArrayList<>();

// Grouping by field
Map<String, List<User>> byDept = users.stream()
    .collect(Collectors.groupingBy(User::getDepartment));

// Partition by predicate
Map<Boolean, List<User>> active = users.stream()
    .collect(Collectors.partitioningBy(User::isActive));

// Collecting to TreeSet (sorted)
Set<String> names = users.stream()
    .map(User::getName)
    .collect(Collectors.toCollection(TreeSet::new));

// Flatten nested collections
List<Task> allTasks = users.stream()
    .flatMap(u -> u.getTasks().stream())
    .toList();
```

### Interview Answer

> "Master the core collection types: ArrayList for sequences, HashSet for uniqueness, HashMap for associations. Know when to use TreeSet/TreeMap (sorted), LinkedHashSet/LinkedHashMap (insertion order). Understand HashMap internals — bucket array, hash function, collisions, treeification.
>
> For type safety, always use generics, never raw types. For concurrent scenarios, use ConcurrentHashMap with atomic methods. Use Streams for transformations — groupingBy, partitioningBy, flatMap. And remember PECS for wildcard constraints: producers use ? extends, consumers use ? super."

---

## Q29. Linked List in a Bucket — put() and get() Internals

### put(key, value) — Step by Step
```java
map.put("Bob", 25);

// JDK Node structure:
static class Node<K,V> {
    final int   hash;      // Cached hash code of key
    final K     key;
    V           value;
    Node<K,V>   next;      // Points to next node in same bucket (chain)
}

// put() logic (pseudo-code):
void put(K key, V value) {
    int hash  = hash(key);
    int index = hash & (table.length - 1);
    Node node = table[index];

    if (node == null) {
        table[index] = new Node(hash, key, value, null);  // First entry in bucket
    } else {
        // Walk the chain (linked list or tree)
        while (node != null) {
            if (node.hash == hash && (node.key == key || key.equals(node.key))) {
                node.value = value;  // KEY FOUND → update value
                return;
            }
            if (node.next == null) {
                node.next = new Node(hash, key, value, null);  // KEY NOT FOUND → append
                if (binCount >= TREEIFY_THRESHOLD) treeifyBin(index); // Maybe treeify
                break;
            }
            node = node.next;
        }
    }

    if (++size > threshold) resize();  // Resize if load factor exceeded
}
```

### get(key) — Step by Step
```java
// get() logic (pseudo-code):
V get(K key) {
    int hash  = hash(key);
    int index = hash & (table.length - 1);
    Node node = table[index];

    while (node != null) {
        // First check hash (cheap int compare), THEN equals (potentially expensive)
        if (node.hash == hash && (node.key == key || key.equals(node.key))) {
            return node.value;  // FOUND
        }
        node = node.next;
    }
    return null;  // NOT FOUND
}
```

### Interview Answer
> "When you put(key, value) into HashMap, the hash function computes a hash code, and a bitwise AND with capacity-1 gives the bucket index. If that bucket is empty, a new Node is created. If it's occupied, the hash code and equals method are used to search the chain. If the key is found (same hash AND equals returns true), the value is updated (put overwrites). If not found, a new Node is appended to the chain. On get(key), the same lookup happens: compute index, then walk the chain comparing hash first (cheap), then equals (potentially expensive). This is why equals and hashCode are critical — if they're inconsistent, keys become unreachable."
>
> *Likely follow-up: "What happens if collision happens and the chain gets very long?"*

---

## Q30. Can We Find Data by Map VALUE?

```java
// HashMap is designed for key → value lookup, NOT value → key
// Finding by value is O(n) — scan all entries

Map<String, Integer> scores = Map.of("Alice", 92, "Bob", 78, "Charlie", 85);

// Find key by value (O(n)):
String nameWith92 = scores.entrySet().stream()
    .filter(e -> e.getValue().equals(92))
    .map(Map.Entry::getKey)
    .findFirst()
    .orElse("Not found");
// "Alice"

// Multiple keys with same value:
List<String> namesWith80Plus = scores.entrySet().stream()
    .filter(e -> e.getValue() >= 80)
    .map(Map.Entry::getKey)
    .toList();
// [Alice, Charlie]

// If you frequently need both directions → use a BiMap (Guava) or maintain two maps:
Map<String, Integer> nameToScore = new HashMap<>();
Map<Integer, String> scoreToName = new HashMap<>();

nameToScore.put("Alice", 92);
scoreToName.put(92, "Alice");  // Must keep in sync on every put/remove
```

### Interview Answer
> "HashMap is fundamentally a key-to-value lookup structure, not the reverse. Finding by value is an O(n) operation — you must scan every entry using entrySet().stream().filter(), comparing each value. There's no index on values. If you frequently need bidirectional lookups, use Guava's BiMap which maintains both key-to-value and value-to-key mappings, or manually maintain two separate HashMaps and keep them in sync on every mutation. For read-heavy scenarios with rare value lookups, the stream approach is fine. For frequent reverse lookups, a second map is worth the extra memory."
>
> *Likely follow-up: "Have you used BiMap before? When would you choose a BiMap over two separate maps?"*

---

## Q31. Traversal and Complexity Summary

| Operation | Best Case | Worst Case (all in one bucket) | Java 8+ Worst Case |
|---|---|---|---|
| `put(key, v)` | O(1) | O(n) LinkedList chain | O(log n) tree |
| `get(key)` | O(1) | O(n) | O(log n) |
| `containsKey` | O(1) | O(n) | O(log n) |
| `containsValue` | O(n) | O(n) | O(n) |
| Iteration (all entries) | O(n + capacity) | O(n + capacity) | O(n + capacity) |

```java
// Why O(n + capacity) for iteration?
// Iterator walks the entire backing array, skipping nulls
// 1000 entries in a capacity-65536 map → lots of empty bucket skips
// Always prefer entrySet().forEach() over keySet() + get() to avoid double lookup:

// BAD: two lookups per entry
for (String key : map.keySet()) {
    Integer val = map.get(key);  // Extra O(1) lookup for each key
}

// GOOD: single pass through entries
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    String key = entry.getKey();
    Integer val = entry.getValue();  // No extra lookup
}
```

### Interview Answer
> "HashMap operations are O(1) average case, O(n) worst case when all keys hash to the same bucket. Java 8 fixed the worst-case by converting long chains (≥8 nodes) to Red-Black trees, making worst-case O(log n). Iteration over a HashMap is O(n + capacity) because the iterator scans the entire backing array, not just the populated buckets — in a map with 1000 entries in a 65536-capacity array, you'll skip thousands of null buckets. Always use entrySet().forEach() to avoid the double-lookup cost of keySet() + get(key). ConcurrentHashMap uses segment-level locking and striped concurrency for thread-safe operations without sacrificing too much throughput."
>
> *Likely follow-up: "Why O(n + capacity) for iteration instead of just O(n)?"*

---

## Q32. Ensure Two `Employee` Objects with Same Name Are NOT Stored as Distinct Keys

### Problem
Without overriding `hashCode` + `equals`, two `Employee` objects with the same name are treated as different keys (default uses memory address).

```java
// BAD: No hashCode/equals overridden
class Employee {
    String name;
    Employee(String name) { this.name = name; }
}

Map<Employee, String> map = new HashMap<>();
Employee e1 = new Employee("Alice");
Employee e2 = new Employee("Alice");

map.put(e1, "Engineer");
map.put(e2, "Manager");

System.out.println(map.size());  // 2! — treats e1 and e2 as different keys
map.get(new Employee("Alice"));  // null! — yet another "different" key
```

```java
// GOOD: Override hashCode and equals based on name
public class Employee {
    private final String name;

    public Employee(String name) {
        this.name = Objects.requireNonNull(name, "name must not be null");
    }

    @Override
    public int hashCode() {
        return Objects.hash(name);     // Same name → same hash → same bucket
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee emp)) return false;
        return Objects.equals(name, emp.name);  // Same name → equal
    }
}

// Now:
Map<Employee, String> map = new HashMap<>();
Employee e1 = new Employee("Alice");
Employee e2 = new Employee("Alice");

map.put(e1, "Engineer");
map.put(e2, "Manager");   // Same key → OVERWRITES, not adds

System.out.println(map.size());                        // 1 ✓
System.out.println(map.get(new Employee("Alice")));    // "Manager" ✓
```

### Using Lombok
```java
// Lombok handles it cleanly — generates correct hashCode + equals based on all fields
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
@RequiredArgsConstructor
public class Employee {
    @EqualsAndHashCode.Include
    private final String name;    // Only name is used for equality
    private String       department;  // excluded — two Alices in different depts = same key
}
```

---

## Quick Revision Card — Topic 6

| Topic | Core Points |
|-------|---|
| **List/Set/Map interfaces** | List=ordered+dupes (ArrayList). Set=unique (HashSet, TreeSet). Map=key→value (HashMap). |
| **ArrayList vs LinkedList** | ArrayList O(1) access, O(1) append. LinkedList O(n) access, O(1) head/tail. Use ArrayList almost always. |
| **Set implementations** | HashSet=O(1), no order. LinkedHashSet=O(1), insertion order. TreeSet=O(log n), sorted. EnumSet=O(1), tiny. |
| **Map implementations** | HashMap=O(1), no order. LinkedHashMap=O(1), insertion/access order. TreeMap=O(log n), sorted. |
| **Comparable vs Comparator** | Comparable=natural order inside class. Comparator=external strategy. Thencomparing for multi-level. |
| **HashMap internals** | Array of buckets. hash()→index. Collision→chain/tree. LoadFactor 0.75→resize double. |
| **hashCode contract** | equals→same hashCode (mandatory). Non-equal can share hash (collision, ok). Use Objects.hash(...). |
| **Collisions & Java 8 fix** | Chain≥8→treeify (Red-Black tree). Threshold=8 (rare), untreeify=6. O(log n) worst case. |
| **Generics with collections** | Type erasure at runtime. Covariance (? extends) for producers. Contravariance (? super) for consumers. PECS rule. |
| **ConcurrentHashMap** | Bucket-level locking, lock-free reads. No nulls. Use putIfAbsent/computeIfAbsent for atomic ops. |

---

## Summary: When to Use Each Collection

| Use Case | Type | Why |
|----------|------|-----|
| **Ordered list, mutable** | ArrayList | Fast random access O(1), fast append O(1) |
| **Unique requirements** | HashSet | Fast O(1) add/remove/contains |
| **Sorted unique set** | TreeSet | Maintains order, fast O(log n) |
| **Key-value pairs** | HashMap | Fast O(1) lookup |
| **Sorted by key** | TreeMap | Maintains order, range queries |
| **Insertion order matters** | LinkedHashSet/LinkedHashMap | O(1) access + order preservation |
| **Thread-safe map** | ConcurrentHashMap | No nulls, bucket-level lock, lock-free reads |
| **Thread-safe list** | CopyOnWriteArrayList | Expensive writes, fast reads, iteration-safe |

---

**End of Topic 6 — Collections Framework, Generics & HashMap Deep-Dive**
