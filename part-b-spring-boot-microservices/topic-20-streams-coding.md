# Topic 20: Streams Coding

## Q145. Pairs Summing to a Target

```java
// input1 = {2, 3, 4, -3, 7, 9, 14, 10}, input2 = 11
// Expected: (2,9), (4,7), (14,-3)

List<Integer> nums = List.of(2, 3, 4, -3, 7, 9, 14, 10);
int target = 11;

Set<Integer> seen = new HashSet<>();
nums.stream()
    .filter(n -> seen.contains(target - n)    // complement already seen?
              || !seen.add(n) == false)       // always add; filter only on complement found
    .peek(n -> seen.add(n))                  // ensure n is in seen even if not filtered
    // Better approach: imperative scan is cleaner for stateful logic
    .forEach(System.out::println);

// Clean solution:
Set<Integer> visited = new HashSet<>();
nums.forEach(n -> {
    if (visited.contains(target - n)) {
        System.out.println("(" + (target - n) + ", " + n + ")");
    }
    visited.add(n);
});
// Output: (2, 9)   (4, 7)   (14, -3)
```

**Interview tip:** Streams with shared mutable state (`seen`) break thread-safety in parallel streams. For pair-finding, hybrid imperative is cleaner. Be ready to explain *why* pure stream isn't always best.

---

## Q146. Flatten Multiple Lists

```java
List<Integer> L1 = List.of(1, 2, 3);
List<Integer> L2 = List.of(4, 5, 6);
List<Integer> L3 = List.of(7, 8, 9);

// flatMap flattens a stream of streams into one stream
List<Integer> merged = Stream.of(L1, L2, L3)
        .flatMap(Collection::stream)
        .collect(Collectors.toList());

// Output: [1, 2, 3, 4, 5, 6, 7, 8, 9]

// Alternative: Stream.concat (only 2 at a time)
List<Integer> merged2 = Stream.concat(
        Stream.concat(L1.stream(), L2.stream()),
        L3.stream()
).collect(Collectors.toList());
```

---

## Q147. Count Male and Female Employees

```java
List<Employee> employees = List.of(
    new Employee(1L, "Alice", "F", "IT", 70000, 30),
    new Employee(2L, "Bob",   "M", "HR", 60000, 28),
    new Employee(3L, "Carol", "F", "IT", 80000, 35),
    new Employee(4L, "Dave",  "M", "IT", 75000, 32)
);

// partitioningBy — splits into true/false buckets
Map<Boolean, Long> byGender = employees.stream()
        .collect(Collectors.partitioningBy(
                e -> e.gender().equals("F"),
                Collectors.counting()
        ));
// {false=2, true=2}  (false=Male, true=Female)

// groupingBy — more flexible, groups by any key
Map<String, Long> genderCount = employees.stream()
        .collect(Collectors.groupingBy(Employee::gender, Collectors.counting()));
// {"F"=2, "M"=2}

// groupingBy with list of names per gender
Map<String, List<String>> namesByGender = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::gender,
                Collectors.mapping(Employee::name, Collectors.toList())
        ));
// {"F"=["Alice","Carol"], "M"=["Bob","Dave"]}
```

**Difference:** `partitioningBy` always returns a `Map<Boolean, …>` (exactly 2 buckets). `groupingBy` returns `Map<K, …>` with as many buckets as distinct keys.

---

## Q148. Highest Salary Per Department

```java
// groupingBy dept + maxBy salary
Map<String, Optional<Employee>> highestPerDept = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::dept,
                Collectors.maxBy(Comparator.comparingDouble(Employee::salary))
        ));

highestPerDept.forEach((dept, emp) ->
        System.out.println(dept + " → " + emp.map(Employee::name).orElse("N/A")));

// Without Optional wrapper:
Map<String, Employee> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::dept,
                e -> e,
                BinaryOperator.maxBy(Comparator.comparingDouble(Employee::salary))
        ));
// BinaryOperator.maxBy as the merge function resolves duplicate keys by keeping the higher salary
```

---

## Q149. Sum of Even and Odd Numbers

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// Using filter
int sumEven = numbers.stream().filter(n -> n % 2 == 0).mapToInt(Integer::intValue).sum();
int sumOdd  = numbers.stream().filter(n -> n % 2 != 0).mapToInt(Integer::intValue).sum();

// Single pass using partitioningBy (no filter, one iteration)
Map<Boolean, Integer> sums = numbers.stream()
        .collect(Collectors.partitioningBy(
                n -> n % 2 == 0,
                Collectors.summingInt(Integer::intValue)
        ));
System.out.println("Even sum: " + sums.get(true));   // 30
System.out.println("Odd sum: "  + sums.get(false));  // 25

// Follow-up: without filter — use partitioningBy or groupingBy(n % 2)
Map<Integer, IntSummaryStatistics> stats = numbers.stream()
        .collect(Collectors.groupingBy(
                n -> n % 2,           // 0 = even, 1 = odd
                Collectors.summarizingInt(Integer::intValue)
        ));
System.out.println("Even sum: " + stats.get(0).getSum());
System.out.println("Odd sum: "  + stats.get(1).getSum());
```

---

## Q150. Find Duplicates

```java
List<Integer> list = List.of(1, 2, 3, 2, 4, 3, 5, 1);

// Approach 1: filter with Set (2 iterations conceptually)
Set<Integer> seen = new HashSet<>();
Set<Integer> duplicates = list.stream()
        .filter(n -> !seen.add(n))   // add() returns false if already present
        .collect(Collectors.toSet());
// {1, 2, 3}

// Approach 2: groupingBy + counting, then filter count > 1
Set<Integer> dups = list.stream()
        .collect(Collectors.groupingBy(n -> n, Collectors.counting()))
        .entrySet().stream()
        .filter(e -> e.getValue() > 1)
        .map(Map.Entry::getKey)
        .collect(Collectors.toSet());

// Minimum iteration: Set.add() approach (Approach 1) — single pass O(n)
```

**Does Stream reduce iteration?** No — streams don't reduce algorithmic complexity. They provide a functional API over the same iteration. Minimum iteration = O(n) with Set.add(), same as imperative loop.

---

## Q151. String Character Operations

```java
String input = "h$e#l!l@o";

// Remove all occurrences of a given char
char toRemove = 'l';
String result = input.chars()
        .filter(c -> c != toRemove)
        .collect(StringBuilder::new, StringBuilder::appendCodePoint, StringBuilder::append)
        .toString();
// "h$e#!@o"  (both 'l' removed)

// Simpler: regex
String noL = input.replaceAll("l", "");

// Find special characters (non-alphanumeric)
List<Character> specials = input.chars()
        .filter(c -> !Character.isLetterOrDigit(c))
        .mapToObj(c -> (char) c)
        .collect(Collectors.toList());
// ['$', '#', '!', '@']

// Remove all special characters
String clean = input.replaceAll("[^a-zA-Z0-9]", "");
// "hello"
```

---

## Q152. Numbers Starting with "1"

```java
List<Integer> nums = List.of(10, 25, 11, 130, 200, 15, 1000);

List<Integer> startWithOne = nums.stream()
        .filter(n -> String.valueOf(n).startsWith("1"))
        .collect(Collectors.toList());
// [10, 11, 130, 15, 1000]

// Using map + filter chain with toString
List<String> asStrings = nums.stream()
        .map(String::valueOf)
        .filter(s -> s.charAt(0) == '1')
        .collect(Collectors.toList());
```

---

## Q153. Employee with Highest Salary

```java
// Option 1: max() with Comparator
Optional<Employee> topEarner = employees.stream()
        .max(Comparator.comparingDouble(Employee::salary));

topEarner.ifPresent(e -> System.out.println(e.name() + ": " + e.salary()));

// Option 2: sorted + findFirst
Optional<Employee> top = employees.stream()
        .sorted(Comparator.comparingDouble(Employee::salary).reversed())
        .findFirst();

// Option 3: reduce
Optional<Employee> top2 = employees.stream()
        .reduce((a, b) -> a.salary() > b.salary() ? a : b);
```

---

## Q154. Concatenate Strings with Delimiter

```java
List<String> cities = List.of("London", "Paris", "Tokyo", "New York");

// Collectors.joining
String result = cities.stream()
        .collect(Collectors.joining(", "));
// "London, Paris, Tokyo, New York"

// With prefix and suffix
String withBrackets = cities.stream()
        .collect(Collectors.joining(", ", "[", "]"));
// "[London, Paris, Tokyo, New York]"

// String.join shortcut (no stream needed)
String joined = String.join(", ", cities);
```

---

## Q155. First Non-Repeating Character

```java
String s = "aabbcdeef";

// LinkedHashMap preserves insertion order; find first with count == 1
Character result = s.chars()
        .mapToObj(c -> (char) c)
        .collect(Collectors.groupingBy(c -> c, LinkedHashMap::new, Collectors.counting()))
        .entrySet().stream()
        .filter(e -> e.getValue() == 1)
        .map(Map.Entry::getKey)
        .findFirst()
        .orElse(null);
// 'c'
```

**Key:** Use `LinkedHashMap::new` as the map factory in `groupingBy` — otherwise the map has no guaranteed order and you can't rely on "first occurrence".

---

## Q156. Character Frequency

```java
String s = "programming";

// char → frequency map
Map<Character, Long> freq = s.chars()
        .mapToObj(c -> (char) c)
        .collect(Collectors.groupingBy(c -> c, Collectors.counting()));
// {p=1, r=2, o=1, g=2, a=1, m=2, i=1, n=1}

// Sorted by frequency descending
freq.entrySet().stream()
        .sorted(Map.Entry.<Character, Long>comparingByValue().reversed())
        .forEach(e -> System.out.println(e.getKey() + " = " + e.getValue()));
// r=2, g=2, m=2, p=1, o=1, a=1, i=1, n=1
```

---

## Q157. Group Strings by Length

```java
List<String> words = List.of("hi", "hey", "hello", "ok", "yes", "world");

Map<Integer, List<String>> byLength = words.stream()
        .collect(Collectors.groupingBy(String::length));
// {2=[hi, ok], 3=[hey, yes], 5=[hello, world]}
```

---

## Q158. Filter > 10 and Find Average

```java
List<Integer> nums = List.of(3, 11, 8, 15, 9, 20, 7, 13);

OptionalDouble avg = nums.stream()
        .filter(n -> n > 10)
        .mapToInt(Integer::intValue)
        .average();
// OptionalDouble[14.75]  (11+15+20+13 = 59, /4 = 14.75)

avg.ifPresent(a -> System.out.printf("Average: %.2f%n", a));
```

---

## Q159. Total Amount Per Item (Transactions)

```java
record Transaction(String itemId, double amount) {}

List<Transaction> txns = List.of(
    new Transaction("A", 100.0),
    new Transaction("B",  50.0),
    new Transaction("A",  75.0),
    new Transaction("C",  30.0),
    new Transaction("B",  20.0)
);

Map<String, Double> totalPerItem = txns.stream()
        .collect(Collectors.groupingBy(
                Transaction::itemId,
                Collectors.summingDouble(Transaction::amount)
        ));
// {A=175.0, B=70.0, C=30.0}
```

---

## Q160. Group by Name, Sum Quantities, Sort Descending

```java
record Item(String name, int quantity) {}

List<Item> items = List.of(
    new Item("Apple", 5),
    new Item("Banana", 3),
    new Item("Apple", 2),
    new Item("Cherry", 8),
    new Item("Banana", 4)
);

Map<String, Integer> sorted = items.stream()
        .collect(Collectors.groupingBy(
                Item::name,
                Collectors.summingInt(Item::quantity)
        ))
        .entrySet().stream()
        .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
        .collect(Collectors.toMap(
                Map.Entry::getKey,
                Map.Entry::getValue,
                (a, b) -> a,
                LinkedHashMap::new  // Preserve sorted order
        ));
// {Cherry=8, Apple=7, Banana=7}
```

**Key:** Regular `HashMap` doesn't preserve order. Use `LinkedHashMap::new` as the map supplier in `toMap` to retain the sorted order.

---

## Q161. Sort Students by Multiple Fields

```java
List<Student> students = List.of(
    new Student(3, "Charlie", "IT", 80),
    new Student(1, "Alice",   "CS", 90),
    new Student(2, "Bob",     "IT", 75),
    new Student(1, "Aaron",   "HR", 85)
);

// Sort by userId ASC, then firstName ASC
List<Student> sorted = students.stream()
        .sorted(Comparator.comparingInt(Student::userId)
                          .thenComparing(Student::firstName))
        .collect(Collectors.toList());
// [userId=1,Aaron], [userId=1,Alice], [userId=2,Bob], [userId=3,Charlie]

// Sort by marks DESC, then name ASC
List<Student> byMarks = students.stream()
        .sorted(Comparator.comparingInt(Student::marks).reversed()
                          .thenComparing(Student::firstName))
        .collect(Collectors.toList());
```

---

## Q162. Department-wise Students with Marks > 50

```java
Map<String, List<Student>> result = students.stream()
        .filter(s -> s.marks() > 50)
        .collect(Collectors.groupingBy(Student::dept));
// {"IT"=[Charlie,Bob], "CS"=[Alice], "HR"=[Aaron]}

// With only names per department
Map<String, List<String>> namesByDept = students.stream()
        .filter(s -> s.marks() > 50)
        .collect(Collectors.groupingBy(
                Student::dept,
                Collectors.mapping(Student::firstName, Collectors.toList())
        ));
```

---

## Q163. Split Employees into Two Lists Without filter()

```java
// Without filter() — use partitioningBy
Map<Boolean, List<Employee>> partitioned = employees.stream()
        .collect(Collectors.partitioningBy(e -> e.gender().equals("F")));

List<Employee> females = partitioned.get(true);
List<Employee> males   = partitioned.get(false);

// Alternative: Collectors.groupingBy
Map<String, List<Employee>> byGender = employees.stream()
        .collect(Collectors.groupingBy(Employee::gender));

List<Employee> femaleList = byGender.getOrDefault("F", List.of());
List<Employee> maleList   = byGender.getOrDefault("M", List.of());
```

---

## Q164. Why Counter Stays 0 Without Terminal Operation

```java
int[] counter = {0};

Stream<Integer> stream = List.of(1, 2, 3, 4, 5).stream()
        .filter(n -> {
            counter[0]++;     // Side effect
            return n % 2 == 0;
        });

System.out.println(counter[0]);  // Prints: 0 ← Nothing executed yet!

// Now add terminal operation
List<Integer> evens = stream.collect(Collectors.toList());
System.out.println(counter[0]);  // Prints: 5 ← Now filter ran
```

**Why:** Streams are **lazy**. Intermediate operations (`filter`, `map`, `sorted`) are not executed until a **terminal operation** (`collect`, `count`, `forEach`, `findFirst`) is called. The stream pipeline is assembled as a description of work; execution happens on demand.

**Implication:** You cannot rely on stream intermediate operations for side effects. Side effects in streams are unreliable — use `forEach` (terminal) instead, or use imperative code.

---

## Q165. reduce() — Product and Total Salary

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5);

// With identity (returns T, never Optional)
int product = nums.stream()
        .reduce(1, (a, b) -> a * b);
// 120

// Without identity (returns Optional<T> — may be empty if stream is empty)
Optional<Integer> productOpt = nums.stream()
        .reduce((a, b) -> a * b);
// Optional[120]

// Total salary of employees
double totalSalary = employees.stream()
        .reduce(0.0,
                (sum, emp) -> sum + emp.salary(),   // accumulator
                Double::sum);                       // combiner (for parallel streams)
// Or simpler:
double total = employees.stream()
        .mapToDouble(Employee::salary)
        .sum();
// 285000.0

// reduce() vs collect():
// reduce(): immutable accumulation (combines values into one)
// collect(): mutable accumulation (builds a collection)
```

**Interview tip:** `reduce(identity, accumulator)` — identity must be a neutral element (0 for sum, 1 for product, "" for concat). If empty stream, returns identity. Without identity, returns `Optional.empty()` for empty stream.

---

## Q166. Collectors.toMap() + Duplicate Key Handling

```java
List<Employee> employees = List.of(
    new Employee(1L, "Alice", "F", "IT", 70000, 30),
    new Employee(2L, "Bob",   "M", "HR", 60000, 28),
    new Employee(3L, "Carol", "F", "IT", 80000, 35)
);

// Basic: id → name (unique keys)
Map<Long, String> idToName = employees.stream()
        .collect(Collectors.toMap(Employee::id, Employee::name));
// {1=Alice, 2=Bob, 3=Carol}

// With duplicate keys — without merge function:
List<Employee> withDup = List.of(
    new Employee(1L, "Alice", "F", "IT", 70000, 30),
    new Employee(1L, "AliceV2", "F", "CS", 75000, 31)  // same id!
);
// This THROWS: IllegalStateException: Duplicate key 1

// With merge function (keep higher salary):
Map<Long, String> safe = withDup.stream()
        .collect(Collectors.toMap(
                Employee::id,
                Employee::name,
                (existing, duplicate) -> existing  // keep first
        ));

// Map<id, Employee> — keep employee with higher salary on conflict
Map<Long, Employee> byId = withDup.stream()
        .collect(Collectors.toMap(
                Employee::id,
                e -> e,
                (e1, e2) -> e1.salary() >= e2.salary() ? e1 : e2
        ));

// Preserving insertion order
Map<Long, String> ordered = employees.stream()
        .collect(Collectors.toMap(
                Employee::id,
                Employee::name,
                (a, b) -> a,
                LinkedHashMap::new  // 4th arg: map factory
        ));
```

---

## Q167. Collectors.joining()

```java
List<String> cities = List.of("London", "Paris", "Tokyo", "New York");

// Simple delimiter
String csv = cities.stream().collect(Collectors.joining(", "));
// "London, Paris, Tokyo, New York"

// With prefix and suffix
String json = cities.stream().collect(Collectors.joining("\", \"", "[\"", "\"]"));
// ["London", "Paris", "Tokyo", "New York"]

// Cleaner brackets:
String bracketed = cities.stream().collect(Collectors.joining(", ", "[", "]"));
// "[London, Paris, Tokyo, New York]"

// Join employee names sorted alphabetically
String names = employees.stream()
        .map(Employee::name)
        .sorted()
        .collect(Collectors.joining(", "));
// "Alice, Bob, Carol, Dave"

// vs String.join (no stream needed for simple cases)
String simple = String.join(", ", cities);
```

---

## Q168. flatMap vs map

```java
// map: one-to-one transformation
// Stream<List<Item>>  ← what you get with map
List<List<Item>> nestedItems = orders.stream()
        .map(Order::items)           // Each order's items as a List
        .collect(Collectors.toList());
// [[Item1a, Item1b], [Item2a], [Item3a, Item3b, Item3c]]

// flatMap: one-to-many, flattens one level
// Stream<Item>  ← what you get with flatMap
List<Item> allItems = orders.stream()
        .flatMap(order -> order.items().stream())  // Unpack each list into individual items
        .collect(Collectors.toList());
// [Item1a, Item1b, Item2a, Item3a, Item3b, Item3c]

// Real example: all item names across all orders
List<String> itemNames = orders.stream()
        .flatMap(order -> order.items().stream())
        .map(Item::name)
        .distinct()
        .sorted()
        .collect(Collectors.toList());

// flatMap with Optional (Java 9+)
Optional<String> opt1 = Optional.of("hello");
Optional<String> result = opt1.flatMap(s -> s.isEmpty() ? Optional.empty() : Optional.of(s.toUpperCase()));
// Optional["HELLO"]

// Analogy:
// map     → takes 1 order, gives back 1 List<Item>     → Stream<List<Item>> (nested)
// flatMap → takes 1 order, unpacks its items           → Stream<Item>       (flat)
```

---

## Q169. anyMatch, allMatch, noneMatch

```java
List<Employee> employees = List.of(
    new Employee(1L, "Alice", "F", "IT", 120000, 30),
    new Employee(2L, "Bob",   "M", "HR",  60000, 28),
    new Employee(3L, "Carol", "F", "IT",  80000, 17)  // underage
);

// anyMatch: returns true if AT LEAST ONE element matches (short-circuits)
boolean anyHighEarner = employees.stream()
        .anyMatch(e -> e.salary() > 100_000);
// true (Alice earns 120,000) — stops at first match

// allMatch: returns true only if ALL elements match (short-circuits on first false)
boolean allAdults = employees.stream()
        .allMatch(e -> e.age() >= 18);
// false (Carol is 17) — stops at Carol

// noneMatch: returns true if NO element matches (short-circuits on first match)
boolean noneNullName = employees.stream()
        .noneMatch(e -> e.name() == null);
// true — no null names

// Empty stream behaviour (important interview question!):
boolean emptyAny  = Stream.empty().anyMatch(e -> true);   // false — no elements, can't match
boolean emptyAll  = Stream.empty().allMatch(e -> false);  // true  — vacuous truth
boolean emptyNone = Stream.empty().noneMatch(e -> true);  // true  — nothing violated

System.out.println(emptyAll);   // true ← common surprise!
```

**Key interview point:** `allMatch` on an **empty stream returns `true`** (vacuous truth — there are no elements to violate the condition). `anyMatch` on empty returns `false`. `noneMatch` on empty returns `true`.

---

## Q170. IntStream.range() and IntStream.rangeClosed()

```java
// range(startInclusive, endExclusive) — like a for-loop
IntStream.range(0, 5).forEach(System.out::print);
// 01234

// rangeClosed(startInclusive, endInclusive) — includes end
IntStream.rangeClosed(1, 5).forEach(System.out::print);
// 12345

// Sum 1 to N
int n = 100;
int sum = IntStream.rangeClosed(1, n).sum();                       // 5050
long sumMath = (long) n * (n + 1) / 2;                            // 5050 (O(1))

// Iterate over a list WITH INDEX (like a for-each with index)
List<String> names = List.of("Alice", "Bob", "Carol");
IntStream.range(0, names.size())
        .forEach(i -> System.out.println(i + ": " + names.get(i)));
// 0: Alice
// 1: Bob
// 2: Carol

// Generate even numbers 0..20
List<Integer> evens = IntStream.rangeClosed(0, 20)
        .filter(n2 -> n2 % 2 == 0)
        .boxed()
        .collect(Collectors.toList());
// [0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

// IntStream vs Stream<Integer>
IntStream.range(1, 5).sum();           // Direct sum — no boxing
Stream.of(1, 2, 3, 4).mapToInt(x->x).sum();  // Unboxed
// Prefer IntStream/LongStream/DoubleStream for numeric operations — avoids boxing overhead
```

---

## Q171. Stream.iterate() and Stream.generate()

```java
// Stream.iterate() — each element depends on previous (Java 8: infinite; Java 9+: can add predicate)
// Fibonacci using iterate
Stream.iterate(new long[]{0, 1}, f -> new long[]{f[1], f[0] + f[1]})
        .limit(10)
        .map(f -> f[0])
        .forEach(System.out::println);
// 0 1 1 2 3 5 8 13 21 34

// Java 9+: iterate with stop condition (like a for-loop)
Stream.iterate(1, n -> n <= 100, n -> n * 2)
        .forEach(System.out::println);
// 1 2 4 8 16 32 64

// Natural numbers
Stream.iterate(1, n -> n + 1)
        .limit(5)
        .collect(Collectors.toList());
// [1, 2, 3, 4, 5]

// Stream.generate() — each element is independently generated (no dependency)
// Generate 5 random UUIDs
Stream.generate(UUID::randomUUID)
        .limit(5)
        .map(UUID::toString)
        .forEach(System.out::println);

// Generate 5 random doubles
Stream.generate(Math::random)
        .limit(5)
        .forEach(System.out::println);

// How do infinite streams work without OOM?
// Lazy evaluation: generate() creates an infinite stream object but
// elements are only computed ON DEMAND. limit(5) stops after 5 elements.
// Without limit() or other short-circuit terminal op → runs forever / OOM.
```

**Key:** Always terminate infinite streams with `limit(n)`, `findFirst()`, `takeWhile()`, or another short-circuiting operation.

---

## Q172. peek() for Debugging

```java
List<Integer> nums = List.of(5, 3, 8, 1, 9, 2, 7);

// peek() is an intermediate operation — doesn't consume the stream
// Use to inspect elements at each pipeline stage
List<Integer> result = nums.stream()
        .peek(n -> System.out.println("Original:  " + n))
        .filter(n -> n > 4)
        .peek(n -> System.out.println("After filter > 4: " + n))
        .sorted()
        .peek(n -> System.out.println("After sorted: " + n))
        .map(n -> n * 10)
        .peek(n -> System.out.println("After ×10: " + n))
        .collect(Collectors.toList());

// Output:
// Original:  5
// After filter > 4: 5
// ...
// After sorted: 5
// After sorted: 7
// After sorted: 8
// After sorted: 9
// After ×10: 50
// After ×10: 70
// After ×10: 80
// After ×10: 90
// Result: [50, 70, 80, 90]

// Critical: peek WITHOUT a terminal operation = nothing runs
Stream<Integer> stream = nums.stream()
        .filter(n -> n > 4)
        .peek(n -> System.out.println("NEVER PRINTS"));
// No terminal operation → pipeline not executed

// peek() pitfall: don't use for mutating state in production
// Use peek() for debugging only — remove before production
```

---

## Q173. Collectors.partitioningBy() with Downstream Collector

```java
// Simple partition
Map<Boolean, List<Employee>> partition = employees.stream()
        .collect(Collectors.partitioningBy(e -> e.salary() > 50_000));
// {true=[Alice,Carol,Dave], false=[Bob]}

// Partition + count per bucket (downstream collector)
Map<Boolean, Long> countByPartition = employees.stream()
        .collect(Collectors.partitioningBy(
                e -> e.salary() > 50_000,
                Collectors.counting()             // downstream
        ));
// {true=3, false=1}

// Partition + average salary per bucket
Map<Boolean, Double> avgSalary = employees.stream()
        .collect(Collectors.partitioningBy(
                e -> e.salary() > 50_000,
                Collectors.averagingDouble(Employee::salary)
        ));
// {true=75000.0, false=60000.0}

// Partition + get just names per bucket
Map<Boolean, List<String>> namesByBucket = employees.stream()
        .collect(Collectors.partitioningBy(
                e -> e.salary() > 50_000,
                Collectors.mapping(Employee::name, Collectors.toList())
        ));
// {true=[Alice, Carol, Dave], false=[Bob]}

// partitioningBy vs groupingBy:
// partitioningBy → always Map<Boolean, downstream>      (2 buckets max)
// groupingBy     → Map<K, downstream>                   (any number of buckets)
// Use partitioningBy when condition is binary (pass/fail, senior/junior, active/inactive)
```

---

## Q174. Collectors.collectingAndThen()

```java
// collectingAndThen(downstream, finisher)
// downstream: the collector to use first
// finisher: a function applied to the result of the downstream

// 1. Collect to UNMODIFIABLE list
List<String> names = employees.stream()
        .map(Employee::name)
        .collect(Collectors.collectingAndThen(
                Collectors.toList(),
                Collections::unmodifiableList
        ));
// names.add("X") → throws UnsupportedOperationException

// 2. Find employee with highest salary wrapped in Optional
Optional<Employee> highestPaid = employees.stream()
        .collect(Collectors.collectingAndThen(
                Collectors.maxBy(Comparator.comparingDouble(Employee::salary)),
                opt -> opt  // already Optional<Employee> — identity
        ));

// More useful: transform the Optional result
String highestPaidName = employees.stream()
        .collect(Collectors.collectingAndThen(
                Collectors.maxBy(Comparator.comparingDouble(Employee::salary)),
                opt -> opt.map(Employee::name).orElse("N/A")
        ));
// "Carol"

// 3. Count and then label
String summary = employees.stream()
        .collect(Collectors.collectingAndThen(
                Collectors.counting(),
                count -> "Total employees: " + count
        ));
// "Total employees: 4"

// 4. Collect to list and immediately sort
List<Employee> sortedImmutable = employees.stream()
        .collect(Collectors.collectingAndThen(
                Collectors.toList(),
                list -> {
                    list.sort(Comparator.comparingDouble(Employee::salary).reversed());
                    return Collections.unmodifiableList(list);
                }
        ));
```

---

## Quick Revision Card — Section 15

| Q | Concept | Key Method(s) |
|---|---|---|
| **Q145** | Pairs summing to target | `Set.add()` short-circuit trick |
| **Q146** | Flatten multiple lists | `Stream.of(L1,L2,L3).flatMap(Collection::stream)` |
| **Q147** | Count M/F employees | `partitioningBy` (boolean) vs `groupingBy` (any key) |
| **Q148** | Highest salary per dept | `groupingBy` + `maxBy` OR `toMap` with `BinaryOperator.maxBy` |
| **Q149** | Even/odd sums | `partitioningBy` + `summingInt` (single pass, no filter) |
| **Q150** | Find duplicates | `!seen.add(n)` filter — O(n) single pass |
| **Q151** | String char ops | `.chars().filter().collect(StringBuilder::new, …)` |
| **Q152** | Numbers starting with "1" | `String.valueOf(n).startsWith("1")` |
| **Q153** | Highest salary employee | `max(Comparator.comparing…)` or `reduce` |
| **Q154** | Join strings | `Collectors.joining(delim, prefix, suffix)` |
| **Q155** | First non-repeating char | `groupingBy` + `LinkedHashMap::new` + `filter count==1` + `findFirst` |
| **Q156** | Char frequency | `groupingBy(c->c, counting())` |
| **Q157** | Group by length | `groupingBy(String::length)` |
| **Q158** | Filter + average | `filter().mapToInt().average()` returns `OptionalDouble` |
| **Q159** | Total amount per item | `groupingBy(itemId, summingDouble(amount))` |
| **Q160** | Sum + sort descending | `groupingBy` → `summingInt` → `entrySet().stream()` → `sorted` → `toMap(LinkedHashMap)` |
| **Q161** | Sort by multiple fields | `Comparator.comparing().thenComparing()` |
| **Q162** | Dept-wise filtered students | `filter` + `groupingBy` |
| **Q163** | Two lists without filter() | `partitioningBy` or `groupingBy` |
| **Q164** | Lazy evaluation | No terminal op = nothing executes |
| **Q165** | `reduce()` | With identity returns T; without identity returns `Optional` |
| **Q166** | `Collectors.toMap()` | Merge function for duplicates; `LinkedHashMap::new` for order |
| **Q167** | `Collectors.joining()` | `joining(delim, prefix, suffix)` |
| **Q168** | `flatMap` vs `map` | `map` = nested streams; `flatMap` = one flat stream |
| **Q169** | `anyMatch/allMatch/noneMatch` | Empty stream: `allMatch`=true, `anyMatch`=false, `noneMatch`=true |
| **Q170** | `IntStream.range/rangeClosed` | Iterate with index; prefer over boxing for numeric work |
| **Q171** | `Stream.iterate/generate` | Infinite streams; always `limit()` or short-circuit terminal |
| **Q172** | `peek()` | Debug-only; does nothing without terminal op |
| **Q173** | `partitioningBy` + downstream | `partitioningBy(pred, Collectors.counting())` |
| **Q174** | `collectingAndThen()` | `toList()` → `unmodifiableList`; `maxBy` → transform Optional |

---

**End of Section 15**

---

## Additional Deep-Dive (Q145-Q174)

### Senior Stream Design Rules

- Keep pipelines side-effect free unless absolutely necessary.
- Avoid parallel streams for IO-bound or stateful operations.
- Prefer readability over one-liner cleverness in interview and production code.

### Real Project Usage

- For heavy data transforms, teams usually benchmark stream vs loop implementations with realistic input sizes before standardizing.
- Bugs frequently come from hidden mutable shared state inside lambdas; enforce stateless mappers/reducers.
