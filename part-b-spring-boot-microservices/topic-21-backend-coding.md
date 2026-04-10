# Topic 21: Backend Coding

## Q175. Reverse a String / Palindrome / Reverse Ignoring Special Characters

💡 **Strategy:** "Two pointers from both ends. For special-char reverse: both pointers skip non-alpha characters before swapping."

### Part A — Reverse a String

```java
// 🔴 Brute Force: StringBuilder or char array
public String reverse(String s) {
    return new StringBuilder(s).reverse().toString();
}

// 🟢 Optimal: Two Pointers (shows understanding of mechanics)
public String reverseManual(String s) {
    char[] chars = s.toCharArray();
    int lo = 0, hi = chars.length - 1;
    while (lo < hi) {
        char tmp = chars[lo];
        chars[lo++] = chars[hi];
        chars[hi--] = tmp;
    }
    return new String(chars);
}
```

### Part B — Palindrome Check

```java
// 🔴 Brute Force: reverse and compare — O(n) time, O(n) space
public boolean isPalindromeBrute(String s) {
    String rev = new StringBuilder(s).reverse().toString();
    return s.equals(rev);
}

// 🟢 Optimal: Two pointers, O(n) time, O(1) space
public boolean isPalindrome(String s) {
    int lo = 0, hi = s.length() - 1;
    while (lo < hi) {
        if (s.charAt(lo++) != s.charAt(hi--)) return false;
    }
    return true;
}
```

### Part C — Reverse Without Moving Special Characters

```java
// Input:  "a,b$c"   → "c,b$a"
// Input:  "Ab,c,de!" → "ed,c,bA!"

public String reverseIgnoreSpecial(String s) {
    char[] chars = s.toCharArray();
    int lo = 0, hi = chars.length - 1;

    while (lo < hi) {
        // Advance lo past non-letter characters
        while (lo < hi && !Character.isLetter(chars[lo])) lo++;
        // Advance hi past non-letter characters
        while (lo < hi && !Character.isLetter(chars[hi])) hi--;

        // Swap the letters
        char tmp = chars[lo];
        chars[lo++] = chars[hi];
        chars[hi--] = tmp;
    }

    return new String(chars);
}
```

**Trace:** `"Ab,c,de!"`
```
lo=0('A'), hi=7('!'→skip) hi=6('e')  → swap('A','e') → "eb,c,dA!"
lo=1('b'), hi=5('d')                  → swap('b','d') → "ed,c,bA!"
lo=2(','→skip) lo=3('c')
hi=4(','→skip) hi=3('c') → lo==hi → stop
Result: "ed,c,bA!" ✓
```

---

## Q176. String Compression

**Problem:** `"aaabbc"` → `"a3b2c1"` | Sorted: `"d3f3c2a2"` → `"a2c2d3f3"`

💡 **Strategy:** "Walk the string tracking a `count`. When char changes, append `char + count` to result."

### 🔴 Brute Force — Naive loop with string concatenation
```java
public String compressBrute(String s) {
    String result = "";               // Bad: O(n²) due to String immutability
    int i = 0;
    while (i < s.length()) {
        char current = s.charAt(i);
        int count = 0;
        while (i < s.length() && s.charAt(i) == current) {
            count++;
            i++;
        }
        result += current + "" + count;   // String concat in loop → O(n²)
    }
    return result;
}
```

### 🟢 Optimal — StringBuilder O(n)
```java
public String compress(String s) {
    if (s == null || s.isEmpty()) return s;

    StringBuilder sb = new StringBuilder();
    int i = 0;

    while (i < s.length()) {
        char current = s.charAt(i);
        int count = 0;

        while (i < s.length() && s.charAt(i) == current) {
            count++;
            i++;
        }
        sb.append(current).append(count);
    }

    return sb.toString();
}
```

### Sorted Frequency Order — using TreeMap
```java
// "aabbbccdd" → "a2b3c2d2"  (sorted by char)
public String compressSorted(String s) {
    // TreeMap keeps keys in natural (alphabetical) order
    Map<Character, Integer> freq = new TreeMap<>();
    for (char c : s.toCharArray()) {
        freq.merge(c, 1, Integer::sum);
    }

    StringBuilder sb = new StringBuilder();
    freq.forEach((ch, count) -> sb.append(ch).append(count));
    return sb.toString();
}
```

### Without Extra Space — In-place approach (conceptual)
```java
// Real "no extra space" compresses WITHIN the same char array.
// Write-pointer 'w' trails read-pointer 'r'. Works only if compressed form ≤ original.
public int compressInPlace(char[] chars) {
    int w = 0;  // write pointer
    int i = 0;  // read pointer

    while (i < chars.length) {
        char current = chars[i];
        int count = 0;
        while (i < chars.length && chars[i] == current) { i++; count++; }

        chars[w++] = current;
        if (count > 1) {
            for (char c : String.valueOf(count).toCharArray()) {
                chars[w++] = c;
            }
        }
    }
    return w;  // new length
}
```

**Complexity:** Time O(n) | Space O(1) in-place, O(n) with StringBuilder

---

## Q177. Second Highest Frequency Fruit

**Problem:** `["apple","banana","apple","cherry","banana","apple"]` → `"banana"` (freq 2, second after "apple" freq 3)

💡 **Strategy:** "Frequency map → sort entries by count descending → pick the second distinct frequency group."

### 🔴 Brute Force — sort the list, count runs
```java
public String secondHighestFrequencyBrute(List<String> fruits) {
    Map<String, Long> freq = fruits.stream()
        .collect(Collectors.groupingBy(f -> f, Collectors.counting()));

    List<Map.Entry<String, Long>> entries = new ArrayList<>(freq.entrySet());
    entries.sort((a, b) -> Long.compare(b.getValue(), a.getValue()));  // desc

    long highestFreq = entries.get(0).getValue();

    // Find first entry with a different (lower) frequency
    for (Map.Entry<String, Long> e : entries) {
        if (e.getValue() < highestFreq) return e.getKey();
    }
    return null;  // No second distinct frequency
}
```

### 🟢 Optimal — LinkedHashMap + TreeMap for clean O(n log n)
```java
public String secondHighestFrequency(List<String> fruits) {
    // Step 1: Count frequencies
    Map<String, Integer> freqMap = new HashMap<>();
    for (String f : fruits) freqMap.merge(f, 1, Integer::sum);

    // Step 2: Group fruits by their frequency (TreeMap = sorted by freq desc)
    TreeMap<Integer, List<String>> byFreq = new TreeMap<>(Comparator.reverseOrder());
    freqMap.forEach((fruit, count) ->
        byFreq.computeIfAbsent(count, k -> new ArrayList<>()).add(fruit));

    // Step 3: Skip highest freq, return first of second-highest
    Iterator<Map.Entry<Integer, List<String>>> it = byFreq.entrySet().iterator();
    it.next();  // skip highest frequency group

    if (!it.hasNext()) return null;
    return it.next().getValue().get(0);  // first fruit in second-highest group
}
```

**Trace:** `["apple"×3, "banana"×2, "cherry"×1]`
```
freqMap:  {apple=3, banana=2, cherry=1}
byFreq:   {3=[apple], 2=[banana], 1=[cherry]}
skip {3=apple} → return "banana" ✓
```

**Complexity:** Time O(n log n) | Space O(n)

---

## Q178. Sort an Array Without Streams or Built-in Sort

**Problem:** `["rj","pj","cd","ab"]` → `["ab","cd","pj","rj"]` using only a `for` loop.

💡 **Strategy:** "Bubble sort is the simplest to write in an interview — compare adjacent pairs, bubble the largest to the end each pass."

### 🔴 Bubble Sort (simplest to write) — O(n²)
```java
public void bubbleSort(String[] arr) {
    int n = arr.length;
    for (int i = 0; i < n - 1; i++) {
        for (int j = 0; j < n - 1 - i; j++) {
            if (arr[j].compareTo(arr[j + 1]) > 0) {  // Wrong order → swap
                String tmp = arr[j];
                arr[j]     = arr[j + 1];
                arr[j + 1] = tmp;
            }
        }
    }
}
```

### 🟢 Selection Sort — O(n²) fewer swaps, cleaner intent
```java
public void selectionSort(String[] arr) {
    for (int i = 0; i < arr.length - 1; i++) {
        int minIdx = i;  // Assume current position has minimum

        // Find the true minimum in the unsorted portion
        for (int j = i + 1; j < arr.length; j++) {
            if (arr[j].compareTo(arr[minIdx]) < 0) {
                minIdx = j;
            }
        }

        // Swap minimum into position i
        if (minIdx != i) {
            String tmp = arr[i];
            arr[i]     = arr[minIdx];
            arr[minIdx] = tmp;
        }
    }
}
```

**Trace:** `["rj","pj","cd","ab"]`
```
Pass i=0: min="ab" at idx=3 → swap with idx=0 → ["ab","pj","cd","rj"]
Pass i=1: min="cd" at idx=2 → swap with idx=1 → ["ab","cd","pj","rj"]
Pass i=2: min="pj" at idx=2 → no swap needed
Result: ["ab","cd","pj","rj"] ✓
```

**Interview tip:** Mention insertion sort is best for nearly sorted data (O(n) best case). For interviews without sort, bubble/selection is acceptable.

---

## Q179. Word Count in a String

**Problem:** `"Java is great and Java is powerful. Streams in Java are powerful"` → `{Java=3, is=2, great=1, ...}`

💡 **Strategy:** "Split on whitespace+punctuation, normalize to lowercase, then `merge` into a frequency map."

### 🔴 Brute Force — manual split with nested loop
```java
public Map<String, Integer> wordCountBrute(String sentence) {
    String[] words = sentence.split("\\s+");
    Map<String, Integer> count = new LinkedHashMap<>();
    for (String word : words) {
        word = word.replaceAll("[^a-zA-Z]", "").toLowerCase();
        if (word.isEmpty()) continue;
        if (count.containsKey(word)) {
            count.put(word, count.get(word) + 1);
        } else {
            count.put(word, 1);
        }
    }
    return count;
}
```

### 🟢 Optimal — merge() + sorted output
```java
public Map<String, Long> wordCount(String sentence) {
    // Split on space and punctuation, lowercase, filter blanks
    return Arrays.stream(sentence.split("[\\s.,!?]+"))
        .map(String::toLowerCase)
        .filter(w -> !w.isEmpty())
        .collect(Collectors.groupingBy(w -> w, Collectors.counting()));
}

// If you want sorted by frequency descending:
public void printWordCountSorted(String sentence) {
    wordCount(sentence).entrySet().stream()
        .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
        .forEach(e -> System.out.println(e.getKey() + " = " + e.getValue()));
}
```

**Alternative using merge():**
```java
public Map<String, Integer> wordCountMerge(String sentence) {
    Map<String, Integer> freq = new LinkedHashMap<>();
    for (String word : sentence.split("[\\s.,!?]+")) {
        freq.merge(word.toLowerCase(), 1, Integer::sum);
    }
    return freq;
}
```

**Complexity:** Time O(n) | Space O(n) — where n = number of words

---

## Q180. Word Wrap

**Problem:** Given a text and `maxWidth`, break text into lines so no line exceeds `maxWidth` characters.

💡 **Strategy:** "Greedy line packing — add words to the current line as long as they fit. When the next word doesn't fit, start a new line."

### 🟢 Greedy Word Wrap O(n)
```java
public List<String> wordWrap(String text, int maxWidth) {
    String[] words = text.split("\\s+");
    List<String> lines = new ArrayList<>();
    StringBuilder currentLine = new StringBuilder();

    for (String word : words) {
        if (word.length() > maxWidth) {
            // Word itself exceeds maxWidth: hard-break it (or throw)
            if (currentLine.length() > 0) {
                lines.add(currentLine.toString().trim());
                currentLine.setLength(0);
            }
            lines.add(word.substring(0, maxWidth));  // simplified: truncate
            continue;
        }

        // Check if adding this word fits: currentLength + 1 (space) + wordLength
        boolean lineEmpty    = currentLine.length() == 0;
        int     lengthIfAdded = currentLine.length() + (lineEmpty ? 0 : 1) + word.length();

        if (lengthIfAdded <= maxWidth) {
            if (!lineEmpty) currentLine.append(' ');
            currentLine.append(word);
        } else {
            // Flush current line, start new one
            lines.add(currentLine.toString());
            currentLine.setLength(0);
            currentLine.append(word);
        }
    }

    if (currentLine.length() > 0) {
        lines.add(currentLine.toString());
    }

    return lines;
}
```

**Demo:**
```java
wordWrap("The quick brown fox jumps over the lazy dog", 10);
// → ["The quick", "brown fox", "jumps over", "the lazy", "dog"]
```

**Complexity:** Time O(n) | Space O(n)

---

## Q181. Validate JSON String Fields

**Problem:** Given `"{ 'name': 'abc', 'email': 'test@x.com', 'id': 1 }"`, return `false` if name or email is missing.

💡 **Strategy:** "Parse with Jackson `ObjectMapper`. Check `has()` or non-null value for required fields. Never use string contains — it's fragile."

### 🔴 Brute Force — String contains (fragile)
```java
public boolean validateBrute(String json) {
    return json.contains("\"name\"") && json.contains("\"email\"");
    // Bad: fails for {"no_name": ..., "email": ...} if "name" appears in a value
}
```

### 🟢 Optimal — Jackson ObjectMapper
```java
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

public boolean validate(String json) {
    // Handle single-quoted JSON by normalising to double quotes
    json = json.replace("'", "\"");

    try {
        ObjectMapper mapper = new ObjectMapper();
        JsonNode node = mapper.readTree(json);

        String name  = node.path("name").asText(null);
        String email = node.path("email").asText(null);

        // Return false if missing or blank
        return name != null  && !name.isBlank()
            && email != null && !email.isBlank();

    } catch (Exception e) {
        return false;  // Invalid JSON
    }
}
```

### Without Jackson — Manual regex approach (acceptable if no libs)
```java
public boolean validateManual(String json) {
    // Extract value of a key using pattern: "key"\s*:\s*"([^"]+)"
    java.util.regex.Pattern p = java.util.regex.Pattern
        .compile("\"(name|email)\"\\s*:\\s*\"([^\"]+)\"");
    java.util.regex.Matcher m = p.matcher(json.replace("'", "\""));

    Set<String> found = new HashSet<>();
    while (m.find()) found.add(m.group(1));

    return found.contains("name") && found.contains("email");
}
```

---

## Q182. Longest Common Subsequence (LCS)

**Problem:** `S1="GTAB"`, `S2="GXTB"` → LCS = `"GTB"` (length 3)

> Subsequence = characters in order, not necessarily contiguous.

💡 **Strategy:** "DP table: if `s1[i]==s2[j]`, `dp[i][j] = dp[i-1][j-1] + 1`. Else `max(dp[i-1][j], dp[i][j-1])`."

### 🔴 Brute Force — Recursive O(2^n)
```java
public int lcsBrute(String s1, String s2, int m, int n) {
    if (m == 0 || n == 0) return 0;
    if (s1.charAt(m - 1) == s2.charAt(n - 1))
        return 1 + lcsBrute(s1, s2, m - 1, n - 1);
    return Math.max(
        lcsBrute(s1, s2, m - 1, n),   // skip from s1
        lcsBrute(s1, s2, m, n - 1)    // skip from s2
    );
}
```

### 🟢 Optimal — DP Bottom-Up O(m×n)
```java
public int lcs(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];   // dp[i][j] = LCS of s1[0..i-1] and s2[0..j-1]

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1] + 1;    // Characters match → extend
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);  // Take best without one
            }
        }
    }

    return dp[m][n];
}

// Also print the actual LCS string:
public String lcsString(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];

    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            dp[i][j] = s1.charAt(i-1) == s2.charAt(j-1)
                ? dp[i-1][j-1] + 1
                : Math.max(dp[i-1][j], dp[i][j-1]);

    // Trace back through DP table to reconstruct
    StringBuilder sb = new StringBuilder();
    int i = m, j = n;
    while (i > 0 && j > 0) {
        if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
            sb.append(s1.charAt(i - 1));
            i--; j--;
        } else if (dp[i - 1][j] > dp[i][j - 1]) {
            i--;
        } else {
            j--;
        }
    }

    return sb.reverse().toString();  // Built backwards, so reverse
}
```

**DP Table for `S1="GTAB"`, `S2="GXTB"`:**
```
     ""  G  X  T  B
""  [ 0, 0, 0, 0, 0 ]
G   [ 0, 1, 1, 1, 1 ]
T   [ 0, 1, 1, 2, 2 ]
A   [ 0, 1, 1, 2, 2 ]
B   [ 0, 1, 1, 2, 3 ]  ← LCS length = 3

Traceback: B matched, A skipped, T matched, X skipped, G matched → "GTB" ✓
```

**Complexity:** Time O(m×n) | Space O(m×n), reducible to O(n)

---

## Q183. Code Review — Refactor dayOfYear Method

**Problem:** Method uses massive `if/else if` chain for day-of-year calculation. Fix it.

💡 **Strategy:** "Replace `if/else if` with a lookup array for cumulative days. Extract leap year logic. Use `boolean` not `Boolean` for primitives."

### 🔴 Bad Code (what the interviewer showed)
```java
// PROBLEM: Rigid, error-prone, not extensible, boolean vs Boolean confusion
int dayOfYear(int month, int dayOfMonth, int year) {
    int day = dayOfMonth;
    if (month == 2) {
        day += 31;
    } else if (month == 3) {
        day += 31 + 28;
    } else if (month == 4) {
        day += 31 + 28 + 31;
    }
    // ... 12 branches total, leap year ignored entirely
    return day;
}
```

**Problems to call out in the interview:**
1. **Magic numbers** — no explanation of where 31, 28 come from
2. **No leap year handling** — February has 29 days in leap years
3. **No input validation** — what if `month=13` or `dayOfMonth=0`?
4. **`Boolean` vs `boolean`** — should use primitive `boolean` for flags; `Boolean` can be null → NPE

### 🟢 Refactored Solution
```java
public int dayOfYear(int month, int dayOfMonth, int year) {
    // Validate inputs
    if (month < 1 || month > 12) throw new IllegalArgumentException("Invalid month: " + month);
    if (dayOfMonth < 1)          throw new IllegalArgumentException("Invalid day: " + dayOfMonth);

    // Cumulative days at start of each month (index 0 unused, 1=Jan starts at 0)
    int[] daysBeforeMonth = { 0, 0, 31, 59, 90, 120, 151, 181, 212, 243, 273, 304, 334 };

    int totalDays = daysBeforeMonth[month] + dayOfMonth;

    // Add 1 if it's a leap year AND we're past February
    if (isLeapYear(year) && month > 2) {
        totalDays += 1;
    }

    return totalDays;
}

// boolean NOT Boolean — primitive, no null risk
private boolean isLeapYear(int year) {
    return (year % 4 == 0 && year % 100 != 0) || (year % 400 == 0);
}
```

**`boolean` vs `Boolean` — when to use which:**
```java
boolean  flag = true;     // Primitive — no null, no unboxing overhead. USE FOR FLAGS.
Boolean  obj  = null;     // Wrapper — nullable, needed for generics/Optional<Boolean>

// Dangerous:
Boolean result = getValue();   // getValue() could return null
if (result) { ... }            // NullPointerException if result is null!

// Safe:
boolean result = getValue();   // Forces non-null at compile time
```

---

## Q184. Sub-array Permutation Check

**Problem:** Does any permutation of array `B` appear as a contiguous subarray in `A`?
`A=[2,1,3,0,4,5]`, `B=[1,2,3]` → `true` (subarray `[2,1,3]` is a permutation of `B`)

💡 **Strategy:** "Sliding window of size `B.length` over `A`. Compare frequency maps of the window and B — if equal, it's a permutation."

### 🔴 Brute Force — O(n × m × m log m)
```java
public boolean containsPermutation(int[] a, int[] b) {
    int[] sortedB = b.clone();
    Arrays.sort(sortedB);

    for (int i = 0; i <= a.length - b.length; i++) {
        int[] window = Arrays.copyOfRange(a, i, i + b.length);
        Arrays.sort(window);
        if (Arrays.equals(window, sortedB)) return true;
    }
    return false;
}
// Sort every window — O(n * m lg m)
```

### 🟢 Optimal — Sliding Window + Frequency Array O(n)
```java
public boolean containsPermutation(int[] a, int[] b) {
    if (b.length > a.length) return false;

    int[] freqB   = new int[256];   // Frequency count for B
    int[] freqWin = new int[256];   // Frequency count for current window

    // Build frequency map for B and the first window of size b.length
    for (int i = 0; i < b.length; i++) {
        freqB[b[i]]++;
        freqWin[a[i]]++;
    }

    if (Arrays.equals(freqB, freqWin)) return true;

    // Slide the window
    for (int i = b.length; i < a.length; i++) {
        freqWin[a[i]]++;               // Add new element on the right
        freqWin[a[i - b.length]]--;    // Remove element that left on the left

        if (Arrays.equals(freqB, freqWin)) return true;
    }

    return false;
}
```

**Trace:** `A=[2,1,3,0,4,5]`, `B=[1,2,3]`
```
freqB:   {1:1, 2:1, 3:1}
Window0: A[0..2]=[2,1,3] → freqWin={1:1,2:1,3:1} == freqB → return true ✓
```

**Complexity:** Time O(n) | Space O(1) — fixed-size freq arrays

---

## Q185. Gas Station (LeetCode 134)

**Problem:** Circular gas stations. `gas=[1,2,3,4,5]`, `cost=[3,4,5,1,2]`. Find starting index to complete circuit, or `-1`.

💡 **Strategy:** "If total `sum(gas) < sum(cost)` → impossible. Otherwise: greedily pick start. Reset start when running tank goes negative."

### 🔴 Brute Force — O(n²)
```java
public int canCompleteCircuit(int[] gas, int[] cost) {
    for (int start = 0; start < gas.length; start++) {
        int tank = 0;
        int count = 0;
        int i = start;
        while (count < gas.length) {
            tank += gas[i] - cost[i];
            if (tank < 0) break;         // Can't continue from this start
            i = (i + 1) % gas.length;
            count++;
        }
        if (count == gas.length) return start;
    }
    return -1;
}
```

### 🟢 Optimal — Greedy Single Pass O(n)
```java
public int canCompleteCircuit(int[] gas, int[] cost) {
    int totalSurplus  = 0;   // Sum of all (gas - cost); if < 0, impossible
    int currentSurplus = 0;  // Running tank for current attempt
    int startIdx       = 0;  // Candidate start station

    for (int i = 0; i < gas.length; i++) {
        int diff = gas[i] - cost[i];
        totalSurplus   += diff;
        currentSurplus += diff;

        if (currentSurplus < 0) {
            // Can't reach station i+1 from current start
            // Reset: try starting from the NEXT station
            startIdx       = i + 1;
            currentSurplus = 0;
        }
    }

    // If total gas < total cost, no solution exists
    return totalSurplus >= 0 ? startIdx : -1;
}
```

**Key insight:** If starting at `s` and we run out at `i`, then none of the stations `s..i` can be a valid start (they had less surplus than `s`).

**Trace:** `gas=[1,2,3,4,5]`, `cost=[3,4,5,1,2]`
```
i=0: diff=-2, total=-2, curr=-2 → reset start=1, curr=0
i=1: diff=-2, total=-4, curr=-2 → reset start=2, curr=0
i=2: diff=-2, total=-6, curr=-2 → reset start=3, curr=0
i=3: diff=+3, total=-3, curr=+3
i=4: diff=+3, total=0,  curr=+6
totalSurplus=0 ≥ 0 → return start=3 ✓
```

**Complexity:** Time O(n) | Space O(1)

---

## Q186. Merge Intervals — Count Free Days

**Problem:** `totalDays=10`, `matches=[[1,3],[2,5],[7,9]]` → how many days have NO matches?

💡 **Strategy:** "Sort intervals by start. Merge overlapping intervals. Free days = total − covered days."

### 🟢 Solution — Sort + Merge O(n log n)
```java
public int countFreeDays(int totalDays, int[][] matches) {
    if (matches.length == 0) return totalDays;

    // Sort intervals by start day
    Arrays.sort(matches, (a, b) -> a[0] - b[0]);

    List<int[]> merged = new ArrayList<>();
    int[] current = matches[0];

    for (int i = 1; i < matches.length; i++) {
        if (matches[i][0] <= current[1]) {
            // Overlapping or adjacent: extend the current interval
            current[1] = Math.max(current[1], matches[i][1]);
        } else {
            // Gap found: save current, start new interval
            merged.add(current);
            current = matches[i];
        }
    }
    merged.add(current);  // Don't forget the last interval

    // Count busy days across merged intervals
    int busyDays = 0;
    for (int[] interval : merged) {
        busyDays += interval[1] - interval[0] + 1;
    }

    return totalDays - busyDays;
}
```

**Trace:** `totalDays=10`, `[[1,3],[2,5],[7,9]]` (sorted)
```
Start: current=[1,3]
[2,5]: 2 ≤ 3 → merge → current=[1,5]
[7,9]: 7 > 5 → save [1,5], current=[7,9]
Save [7,9]
Merged: [[1,5],[7,9]]
Busy: (5-1+1) + (9-7+1) = 5 + 3 = 8
Free: 10 - 8 = 2 ✓ (days 6 and 10)
```

**Complexity:** Time O(n log n) | Space O(n)

---

## Q187. Minimum Operations to Exceed K (LeetCode 3066)

**Problem:** `nums=[2,3,6,8], k=6`. Repeatedly take 2 smallest: `newVal = min*2 + max`. Count operations until all values ≥ k.

💡 **Strategy:** "Min-Heap (PriorityQueue): always extract the two smallest, combine, re-insert. Count until heap's minimum ≥ k."

### 🟢 Solution — MinHeap O(n log n)
```java
public int minOperations(int[] nums, int k) {
    // Min-heap: smallest element always at top
    PriorityQueue<Long> minHeap = new PriorityQueue<>();
    for (int n : nums) minHeap.offer((long) n);

    int operations = 0;

    while (minHeap.peek() < k) {
        if (minHeap.size() < 2) return -1;   // Not enough elements to combine

        long min1 = minHeap.poll();  // Smallest
        long min2 = minHeap.poll();  // Second smallest

        long combined = min1 * 2 + min2;     // Formula: min*2 + max
        minHeap.offer(combined);

        operations++;
    }

    return operations;
}
```

**Trace:** `nums=[2,3,6,8], k=6`
```
Heap: [2,3,6,8] — peek=2 < 6, operate
  min1=2, min2=3 → combined=2*2+3=7 → heap=[6,7,8]  ops=1
Heap: [6,7,8] — peek=6 ≥ 6, STOP
Return 1 ✓
```

**Another example:** `nums=[2,3,4], k=8`
```
min1=2, min2=3 → combined=7 → heap=[4,7]   ops=1
min1=4, min2=7 → combined=15 → heap=[15]   ops=2
peek=15 ≥ 8, return 2 ✓
```

**Complexity:** Time O(n log n) — each operation is O(log n) heap ops

---

## Q188. SQL — Remove Duplicate Rows, Keep One

**Problem:** Table `employees(id, name, email)` has duplicate rows. Delete duplicates, keep the one with the smallest `id`.

💡 **Strategy:** "Self-join or subquery: keep the MIN(id) per email group. Delete everything else."

### Method 1 — Subquery with MIN (most portable)
```sql
DELETE FROM employees
WHERE id NOT IN (
    SELECT MIN(id)
    FROM employees
    GROUP BY email    -- Group by the column that defines "duplicate"
);
```

### Method 2 — Self-JOIN (MySQL friendly)
```sql
DELETE e1
FROM employees e1
INNER JOIN employees e2
    ON e1.email = e2.email
    AND e1.id > e2.id;    -- Delete e1 when a "better" (lower id) duplicate e2 exists
```

### Method 3 — ROW_NUMBER() (modern SQL: PostgreSQL, SQL Server, Oracle)
```sql
WITH ranked AS (
    SELECT id,
           ROW_NUMBER() OVER (PARTITION BY email ORDER BY id ASC) AS rn
    FROM employees
)
DELETE FROM employees
WHERE id IN (
    SELECT id FROM ranked WHERE rn > 1  -- Keep rn=1, delete rn=2,3,...
);
```

**Interview tip:** Always clarify "which row to keep?" — lowest id, latest timestamp, or any? Mention that `ROW_NUMBER()` is the cleanest, most intention-revealing solution for modern databases.

---

## Q189. Unit Test with Mockito

**Problem:** Write a unit test for a `UserService.getUserById(Long id)` that calls a `UserRepository`.

💡 **Strategy:** "Arrange → Act → Assert (AAA). Mock all dependencies. Test the happy path, empty result, and exception path."

### Service Under Test
```java
@Service
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public UserDTO getUserById(Long id) {
        return userRepository.findById(id)
            .map(user -> new UserDTO(user.getId(), user.getName()))
            .orElseThrow(() -> new ResourceNotFoundException("User not found: " + id));
    }
}
```

### Unit Test Class
```java
@ExtendWith(MockitoExtension.class)        // JUnit 5: replaces @RunWith(MockitoJUnitRunner.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;  // Mockito-created mock — no real DB

    @InjectMocks
    private UserService userService;        // Injects the mock above into UserService

    // ─── Happy Path ───────────────────────────────────────────────
    @Test
    void getUserById_whenUserExists_returnsDTO() {
        // ARRANGE
        User mockUser = new User(1L, "Alice");
        when(userRepository.findById(1L)).thenReturn(Optional.of(mockUser));

        // ACT
        UserDTO result = userService.getUserById(1L);

        // ASSERT
        assertNotNull(result);
        assertEquals(1L,      result.getId());
        assertEquals("Alice", result.getName());

        verify(userRepository, times(1)).findById(1L);  // Ensure repo was called exactly once
    }

    // ─── Not Found Path ───────────────────────────────────────────
    @Test
    void getUserById_whenUserNotFound_throwsException() {
        // ARRANGE
        when(userRepository.findById(99L)).thenReturn(Optional.empty());

        // ACT + ASSERT
        ResourceNotFoundException ex = assertThrows(
            ResourceNotFoundException.class,
            () -> userService.getUserById(99L)
        );

        assertTrue(ex.getMessage().contains("99"));
        verify(userRepository).findById(99L);
    }

    // ─── Repository Throws Exception ──────────────────────────────
    @Test
    void getUserById_whenRepositoryThrows_propagatesException() {
        // ARRANGE
        when(userRepository.findById(anyLong()))
            .thenThrow(new DataAccessException("DB down") {});

        // ACT + ASSERT
        assertThrows(DataAccessException.class, () -> userService.getUserById(1L));
    }

    // ─── Argument Captor (bonus — verify what was passed) ─────────
    @Test
    void getUserById_verifyCorrectIdPassedToRepo() {
        ArgumentCaptor<Long> idCaptor = ArgumentCaptor.forClass(Long.class);
        when(userRepository.findById(idCaptor.capture())).thenReturn(Optional.empty());

        try { userService.getUserById(42L); } catch (Exception ignored) {}

        assertEquals(42L, idCaptor.getValue());   // Confirm exact value reached repo
    }
}
```

**Controller test with MockMvc:**
```java
@WebMvcTest(UserController.class)           // Only loads web layer, not full context
class UserControllerTest {

    @Autowired MockMvc mockMvc;

    @MockBean UserService userService;      // @MockBean = Spring-managed mock

    @Test
    void getUser_returns200() throws Exception {
        when(userService.getUserById(1L)).thenReturn(new UserDTO(1L, "Alice"));

        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"));
    }

    @Test
    void getUser_whenNotFound_returns404() throws Exception {
        when(userService.getUserById(99L)).thenThrow(new ResourceNotFoundException("not found"));

        mockMvc.perform(get("/users/99"))
            .andExpect(status().isNotFound());
    }
}
```

**Key annotations:**

| Annotation | Purpose |
|---|---|
| `@ExtendWith(MockitoExtension.class)` | Enable Mockito in JUnit 5 (no Spring context) |
| `@Mock` | Create a pure Mockito mock |
| `@InjectMocks` | Create instance, inject all `@Mock` fields |
| `@Spy` | Wrap real object — partial mock |
| `@WebMvcTest` | Slice test — only web layer wired up |
| `@MockBean` | Spring-managed mock — enters Spring context |
| `@SpringBootTest` | Full integration test — starts entire context |

---

## Master Cheat Sheet — Section 16 Patterns

| Q | Problem | Pattern | One-Liner Tip |
|---|---|---|---|
| **Q175** | String Reverse / Palindrome | Two Pointers | For special chars: both pointers skip non-letter before swap |
| **Q176** | String Compression | Count runs | Use `StringBuilder`, not `+=`. `TreeMap` for sorted output |
| **Q177** | Second Highest Frequency | Freq Map + TreeMap | Invert: `TreeMap<count, [fruits]>` → skip top, take next |
| **Q178** | Sort Without Built-ins | Selection/Bubble Sort | Selection = fewer swaps, cleaner; Bubble = simpler to write |
| **Q179** | Word Count | HashMap + merge() | `freq.merge(word, 1, Integer::sum)` — idiomatic one-liner |
| **Q180** | Word Wrap | Greedy line packing | Add words while they fit; flush line when next word overflows |
| **Q181** | JSON Validation | Jackson / Regex | Never use `String.contains()` — field name might appear in a value |
| **Q182** | LCS | 2D DP table | `dp[i][j] = (chars match) ? dp[i-1][j-1]+1 : max(up, left)` |
| **Q183** | Code Review refactor | Lookup array + guards | Replace if/else chain with `daysBeforeMonth[]` array |
| **Q184** | Subarray Permutation | Sliding Window + freq[] | Fixed-size window: add right, remove left, compare freq arrays |
| **Q185** | Gas Station | Greedy reset | Reset start when tank < 0. If totalSurplus ≥ 0, answer exists |
| **Q186** | Merge Intervals | Sort + Merge | Sort by start → merge overlaps → total − busyDays = free days |
| **Q187** | Min Operations K | Min-Heap | PriorityQueue: poll 2, combine (min×2+max), re-offer, count |
| **Q188** | SQL Duplicates | ROW_NUMBER() / MIN(id) | Keep `MIN(id)` per group; delete everything else |
| **Q189** | Unit Test / Mockito | AAA pattern | Arrange (mock) → Act → Assert. Test happy + not-found + error paths |

---

**End of Section 16**

---

## Additional Deep-Dive (Q175-Q189)

### Interview Execution Strategy

1. State brute-force first to show completeness.
2. Optimize only after identifying bottleneck clearly.
3. Explain complexity in terms of dominant operation count, not memorized symbols.
4. Call out edge cases before coding (null, empty, duplicates, overflow).

### Real Project Usage

- In backend services, clean error handling and testability often matter more than micro-optimizations unless profile data proves a hotspot.
- For coding interviews, narrating trade-offs clearly is often the differentiator at senior levels.
