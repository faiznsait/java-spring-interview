# Topic 22: DSA

## Q190. Valid Parentheses

**Problem:** `"()[]{}"` → `true` | `"([)]"` → `false` | `"{[]}"` → `true`

💡 **Strategy:** "Push open brackets in; when you see a closing bracket, the top of the stack must be its matching opener."

### 🔴 Brute Force — O(n²)
```java
// Keep replacing "()" "[]" "{}" until nothing changes, then check if empty
public boolean isValid(String s) {
    while (s.contains("()") || s.contains("[]") || s.contains("{}")) {
        s = s.replace("()", "")
             .replace("[]", "")
             .replace("{}", "");
    }
    return s.isEmpty();
}
// Works but O(n²) — each replace scans and rebuilds the string
```

### 🟢 Optimal — Stack O(n)
```java
public boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();

    for (char c : s.toCharArray()) {
        // Push the EXPECTED closing bracket onto the stack
        if      (c == '(') stack.push(')');
        else if (c == '[') stack.push(']');
        else if (c == '{') stack.push('}');
        else {
            // c is a closing bracket
            // Stack must be non-empty AND top must match
            if (stack.isEmpty() || stack.pop() != c) return false;
        }
    }

    return stack.isEmpty(); // All openers must have been matched
}
```

**Trace:** `"({[]})"` → push `)` push `}` push `]` → see `]` pop `]` ✅ → see `}` pop `}` ✅ → see `)` pop `)` ✅ → stack empty → `true`

**Edge cases:** `"]"` → closing bracket with empty stack → false. `"(("` → stack not empty at end → false.

**Complexity:** Time O(n) | Space O(n)

---

## Q191. Best Time to Buy and Sell Stock

**Problem:** `prices = [7,1,5,3,6,4]` → `5` (buy at 1, sell at 6)

💡 **Strategy:** "Track the minimum price seen so far. At each day, profit = today − min. Keep max profit."

### 🔴 Brute Force — O(n²)
```java
public int maxProfit(int[] prices) {
    int maxProfit = 0;
    for (int buy = 0; buy < prices.length; buy++) {
        for (int sell = buy + 1; sell < prices.length; sell++) {
            int profit = prices[sell] - prices[buy];
            maxProfit = Math.max(maxProfit, profit);
        }
    }
    return maxProfit;
}
// Try every (buy, sell) pair — TLE on large input
```

### 🟢 Optimal — Single Pass Greedy O(n)
```java
public int maxProfit(int[] prices) {
    int minPrice  = Integer.MAX_VALUE;
    int maxProfit = 0;

    for (int price : prices) {
        if (price < minPrice) {
            minPrice = price;           // Found a cheaper buy day
        } else {
            int profit = price - minPrice;
            maxProfit = Math.max(maxProfit, profit);  // Best sell today?
        }
    }

    return maxProfit;
}
```

**Trace:** `[7,1,5,3,6,4]`
```
price=7: min=7, profit=0
price=1: min=1  (cheaper buy!)
price=5: profit=5-1=4, maxProfit=4
price=3: profit=3-1=2, maxProfit=4
price=6: profit=6-1=5, maxProfit=5 ✓
price=4: profit=4-1=3, maxProfit=5
```

**Complexity:** Time O(n) | Space O(1)

---

## Q192. Longest Substring Without Repeating Characters

**Problem:** `"abcabcbb"` → `3` (`"abc"`) | `"bbbbb"` → `1` | `"pwwkew"` → `3` (`"wke"`)

💡 **Strategy:** "Sliding window — expand right. When duplicate found, shrink from left until duplicate removed."

### 🔴 Brute Force — O(n²)
```java
public int lengthOfLongestSubstring(String s) {
    int maxLen = 0;
    for (int i = 0; i < s.length(); i++) {
        Set<Character> seen = new HashSet<>();
        for (int j = i; j < s.length(); j++) {
            if (seen.contains(s.charAt(j))) break;  // Duplicate found, stop
            seen.add(s.charAt(j));
            maxLen = Math.max(maxLen, j - i + 1);
        }
    }
    return maxLen;
}
// Check every starting position — O(n²)
```

### 🟢 Optimal — Sliding Window + HashMap O(n)
```java
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastIndex = new HashMap<>(); // char → last seen index
    int maxLen = 0;
    int left = 0;  // Left boundary of window

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);

        // If char seen AND its last position is inside the current window
        if (lastIndex.containsKey(c) && lastIndex.get(c) >= left) {
            left = lastIndex.get(c) + 1;  // Jump left past the duplicate
        }

        lastIndex.put(c, right);                          // Update last seen index
        maxLen = Math.max(maxLen, right - left + 1);      // Window size
    }

    return maxLen;
}
```

**Trace:** `"abcabcbb"`
```
right=0 'a': window=[a],       left=0, max=1
right=1 'b': window=[ab],      left=0, max=2
right=2 'c': window=[abc],     left=0, max=3
right=3 'a': 'a' last at 0 ≥ left=0 → left=1, window=[bca], max=3
right=4 'b': 'b' last at 1 ≥ left=1 → left=2, window=[cab], max=3
right=5 'c': 'c' last at 2 ≥ left=2 → left=3, window=[abc], max=3
right=6 'b': 'b' last at 4 ≥ left=3 → left=5, window=[cb],  max=3
right=7 'b': 'b' last at 6 ≥ left=5 → left=7, window=[b],   max=3
```

**Complexity:** Time O(n) | Space O(min(n, charset_size))

---

## Q193. Kadane's Algorithm — Maximum Subarray Sum

**Problem:** `[-2,1,-3,4,-1,2,1,-5,4]` → `6` (subarray `[4,-1,2,1]`)

💡 **Strategy:** "At each element, decide: extend current subarray OR start fresh here. Keep running max."

### 🔴 Brute Force — O(n²)
```java
public int maxSubArray(int[] nums) {
    int maxSum = Integer.MIN_VALUE;
    for (int i = 0; i < nums.length; i++) {
        int currentSum = 0;
        for (int j = i; j < nums.length; j++) {
            currentSum += nums[j];
            maxSum = Math.max(maxSum, currentSum);
        }
    }
    return maxSum;
}
// Try every subarray starting at each index
```

### 🟢 Optimal — Kadane's O(n)
```java
public int maxSubArray(int[] nums) {
    int currentSum = nums[0];   // Best sum ending at current position
    int maxSum     = nums[0];   // Global maximum

    for (int i = 1; i < nums.length; i++) {
        // Either extend previous subarray, or start fresh from here
        currentSum = Math.max(nums[i], currentSum + nums[i]);
        maxSum = Math.max(maxSum, currentSum);
    }

    return maxSum;
}
```

**Trace:** `[-2,1,-3,4,-1,2,1,-5,4]`
```
i=0: curr=-2,             max=-2
i=1: curr=max(1,-2+1)=1,  max=1
i=2: curr=max(-3,1-3)=-2, max=1
i=3: curr=max(4,-2+4)=4,  max=4
i=4: curr=max(-1,4-1)=3,  max=4
i=5: curr=max(2,3+2)=5,   max=5
i=6: curr=max(1,5+1)=6,   max=6 ✓
i=7: curr=max(-5,6-5)=1,  max=6
i=8: curr=max(4,1+4)=5,   max=6
```

**Insight:** `currentSum = Math.max(nums[i], currentSum + nums[i])` is the same as:
```java
if (currentSum < 0) currentSum = nums[i];   // Negative running sum? Restart
else                currentSum += nums[i];   // Positive? Extend
```

**Complexity:** Time O(n) | Space O(1)

---

## Q194. Subarray Sum Equals K

**Problem:** `nums=[1,1,1], k=2` → `2` | `nums=[1,2,3], k=3` → `2`

💡 **Strategy:** "Prefix sum trick — `sum[i..j] = prefixSum[j] - prefixSum[i-1]`. So look for how many times `(currentSum - k)` has appeared before."

### 🔴 Brute Force — O(n²)
```java
public int subarraySum(int[] nums, int k) {
    int count = 0;
    for (int i = 0; i < nums.length; i++) {
        int sum = 0;
        for (int j = i; j < nums.length; j++) {
            sum += nums[j];
            if (sum == k) count++;
        }
    }
    return count;
}
// Compute sum of every subarray — O(n²)
```

### 🟢 Optimal — Prefix Sum + HashMap O(n)
```java
public int subarraySum(int[] nums, int k) {
    // Key:   prefix sum value
    // Value: how many times this prefix sum has occurred
    Map<Integer, Integer> prefixCount = new HashMap<>();
    prefixCount.put(0, 1);  // Empty subarray has sum 0 (handles subarray from index 0)

    int count = 0;
    int currentSum = 0;

    for (int num : nums) {
        currentSum += num;

        // If (currentSum - k) exists in map, then subarray [that point + 1 .. here] sums to k
        int needed = currentSum - k;
        count += prefixCount.getOrDefault(needed, 0);

        // Record this prefix sum
        prefixCount.put(currentSum, prefixCount.getOrDefault(currentSum, 0) + 1);
    }

    return count;
}
```

**Mental model:**
```
nums = [1, 2, 1, 3], k = 3

prefixSums: 0, 1, 3, 4, 7
            ↑  ↑  ↑  ↑  ↑
         empty 1  1+2 ...

At currentSum=3: needed=3-3=0 → map has {0:1} → count += 1  (subarray [1,2])
At currentSum=4: needed=4-3=1 → map has {1:1} → count += 1  (subarray [2,1])
At currentSum=7: needed=7-3=4 → map has {4:1} → count += 1  (subarray [3])
Total = 3
```

**Complexity:** Time O(n) | Space O(n)

---

## Q195. Contains Duplicate

**Problem:** `[1,2,3,1]` → `true` | `[1,2,3,4]` → `false`

💡 **Strategy:** "HashSet add() returns false if element already exists. First false = duplicate found."

### 🔴 Brute Force — O(n²)
```java
public boolean containsDuplicate(int[] nums) {
    for (int i = 0; i < nums.length; i++)
        for (int j = i + 1; j < nums.length; j++)
            if (nums[i] == nums[j]) return true;
    return false;
}
```

### 🟢 Optimal — HashSet O(n)
```java
public boolean containsDuplicate(int[] nums) {
    Set<Integer> seen = new HashSet<>();
    for (int n : nums) {
        if (!seen.add(n)) return true;  // add() returns false if already present
    }
    return false;
}

// One-liner (slightly slower due to stream overhead):
public boolean containsDuplicateStream(int[] nums) {
    return nums.length != Arrays.stream(nums).distinct().count();
}
```

**Complexity:** Time O(n) | Space O(n)

---

## Q196. Missing Number in Array

**Problem:** `[3,0,1]` → `2` | `[9,6,4,2,3,5,7,0,1]` → `8`

💡 **Strategy for Math:** "Expected sum of 0..n is n*(n+1)/2. Subtract actual sum. The difference is the missing number."
💡 **Strategy for XOR:** "XOR of same numbers cancels out. XOR all indices (0..n) with all array values — only the missing number survives."

### 🔴 Brute Force — O(n log n)
```java
public int missingNumber(int[] nums) {
    Arrays.sort(nums);                           // Sort first
    for (int i = 0; i < nums.length; i++) {
        if (nums[i] != i) return i;              // Expected i, got something else
    }
    return nums.length;                          // Missing the last number
}
```

### 🟢 Optimal 1 — Math O(n)
```java
public int missingNumber(int[] nums) {
    int n = nums.length;
    int expectedSum = n * (n + 1) / 2;  // Sum of 0 + 1 + 2 + ... + n
    int actualSum   = 0;
    for (int num : nums) actualSum += num;
    return expectedSum - actualSum;
}
```

### 🟢 Optimal 2 — XOR O(n) (no overflow risk for very large n)
```java
public int missingNumber(int[] nums) {
    int xor = nums.length;              // Start with n (it's the "extra" index)
    for (int i = 0; i < nums.length; i++) {
        xor ^= i ^ nums[i];            // XOR each index AND each value
    }
    return xor;
    // XOR cancels pairs: 0^0=0, 1^1=0. Only the missing number is unpaired.
}
```

**XOR trace:** `[3,0,1]`, n=3
```
Start: xor = 3
i=0: xor ^= 0 ^ 3 = 3^0^3 = 0
i=1: xor ^= 1 ^ 0 = 0^1^0 = 1
i=2: xor ^= 2 ^ 1 = 1^2^1 = 2  ← answer!
```

**Complexity:** Time O(n) | Space O(1)

---

## Q197. Intersection of Two Arrays

**Problem:** `[1,2,2,1]` and `[2,2]` → `[2]` | `[4,9,5]` and `[9,4,9,8,4]` → `[4,9]`

💡 **Strategy:** "Put array1 in a Set. Walk array2 — if element is in the Set, it's in the intersection."

### 🔴 Brute Force — O(n×m)
```java
public int[] intersection(int[] nums1, int[] nums2) {
    Set<Integer> result = new HashSet<>();
    for (int a : nums1)
        for (int b : nums2)
            if (a == b) result.add(a);
    return result.stream().mapToInt(Integer::intValue).toArray();
}
```

### 🟢 Optimal — HashSet O(n + m)
```java
public int[] intersection(int[] nums1, int[] nums2) {
    Set<Integer> set1  = new HashSet<>();
    Set<Integer> inter = new HashSet<>();

    for (int n : nums1) set1.add(n);           // Load nums1 into set

    for (int n : nums2) {
        if (set1.contains(n)) inter.add(n);    // Found in both
    }

    // Convert Set to int[]
    return inter.stream().mapToInt(Integer::intValue).toArray();
}
```

**Variant – Include duplicates (Intersection II):**
```java
// nums1=[1,2,2,1] nums2=[2,2] → [2,2]
public int[] intersect(int[] nums1, int[] nums2) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums1) freq.merge(n, 1, Integer::sum);  // Count each value

    List<Integer> result = new ArrayList<>();
    for (int n : nums2) {
        if (freq.getOrDefault(n, 0) > 0) {
            result.add(n);
            freq.merge(n, -1, Integer::sum);  // Consume one occurrence
        }
    }
    return result.stream().mapToInt(Integer::intValue).toArray();
}
```

**Complexity:** Time O(n + m) | Space O(n)

---

## Q198. Move Zeroes

**Problem:** `[0,1,0,3,12]` → `[1,3,12,0,0]` (in-place, maintain relative order)

💡 **Strategy:** "Two pointers — `left` chases non-zero values. Move non-zeros to `left`, fill rest with 0s."

### 🔴 Brute Force — O(n) extra space
```java
public void moveZeroes(int[] nums) {
    int[] temp = new int[nums.length];
    int idx = 0;
    for (int n : nums) if (n != 0) temp[idx++] = n;  // Copy non-zeros
    System.arraycopy(temp, 0, nums, 0, nums.length);  // Copy back (zeros auto-filled)
}
```

### 🟢 Optimal — Two Pointers In-Place O(n)
```java
public void moveZeroes(int[] nums) {
    int left = 0;  // Points to the next position for a non-zero element

    // Pass 1: move all non-zeros to the front, in order
    for (int right = 0; right < nums.length; right++) {
        if (nums[right] != 0) {
            nums[left++] = nums[right];
        }
    }

    // Pass 2: fill remaining positions with 0
    while (left < nums.length) {
        nums[left++] = 0;
    }
}
```

**Trace:** `[0,1,0,3,12]`
```
right=0: 0 → skip          left=0
right=1: 1 → nums[0]=1     left=1   [1,1,0,3,12]
right=2: 0 → skip          left=1
right=3: 3 → nums[1]=3     left=2   [1,3,0,3,12]
right=4: 12→ nums[2]=12    left=3   [1,3,12,3,12]
Fill zeros: nums[3]=0, nums[4]=0    [1,3,12,0,0] ✓
```

**Complexity:** Time O(n) | Space O(1)

---

## Q199. Rotate Array

**Problem:** `[1,2,3,4,5,6,7]`, k=3 → `[5,6,7,1,2,3,4]`

💡 **Strategy (Reverse Trick):** "Reverse all → reverse first k → reverse rest. Think of it as: last k elements come first."

### 🔴 Brute Force — O(n×k)
```java
public void rotate(int[] nums, int k) {
    k = k % nums.length;
    for (int i = 0; i < k; i++) {
        int last = nums[nums.length - 1];
        // Shift everything right by 1
        for (int j = nums.length - 1; j > 0; j--) {
            nums[j] = nums[j - 1];
        }
        nums[0] = last;
    }
}
// Rotate by 1, k times — O(n*k)
```

### 🟢 Optimal — Reverse Trick O(n)
```java
public void rotate(int[] nums, int k) {
    k = k % nums.length;              // Handle k > length (e.g., k=10 on length 7 = k=3)

    reverse(nums, 0, nums.length - 1);  // Step 1: reverse entire array
    reverse(nums, 0, k - 1);            // Step 2: reverse first k elements
    reverse(nums, k, nums.length - 1);  // Step 3: reverse remaining elements
}

private void reverse(int[] nums, int lo, int hi) {
    while (lo < hi) {
        int temp = nums[lo];
        nums[lo++] = nums[hi];
        nums[hi--] = temp;
    }
}
```

**Trace:** `[1,2,3,4,5,6,7]`, k=3
```
Step 1 - Reverse all:    [7,6,5,4,3,2,1]
Step 2 - Reverse [0..2]: [5,6,7,4,3,2,1]
Step 3 - Reverse [3..6]: [5,6,7,1,2,3,4] ✓
```

**Complexity:** Time O(n) | Space O(1)

---

## Q200. Product of Array Except Self

**Problem:** `[1,2,3,4]` → `[24,12,8,6]` | No division allowed.

💡 **Strategy:** "For each position, product of everything to the LEFT × product of everything to the RIGHT."

### 🔴 Brute Force — O(n²)
```java
public int[] productExceptSelf(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    for (int i = 0; i < n; i++) {
        int product = 1;
        for (int j = 0; j < n; j++) {
            if (j != i) product *= nums[j];
        }
        result[i] = product;
    }
    return result;
}
```

### 🟢 Optimal — Prefix × Suffix Products O(n)
```java
public int[] productExceptSelf(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];

    // Pass 1 (left to right): result[i] = product of all elements to LEFT of i
    result[0] = 1;
    for (int i = 1; i < n; i++) {
        result[i] = result[i - 1] * nums[i - 1];
    }

    // Pass 2 (right to left): multiply by product of all elements to RIGHT of i
    int rightProduct = 1;
    for (int i = n - 1; i >= 0; i--) {
        result[i] *= rightProduct;
        rightProduct *= nums[i];   // Update for next position to the left
    }

    return result;
}
```

**Visual:**
```
nums:         [1,  2,  3,  4]

Left products: [1,  1,  2,  6]   (1, 1, 1×2, 1×2×3)
Right products:[24, 12,  4,  1]  (2×3×4, 3×4, 4, 1)

result = left × right:
         [1×24, 1×12, 2×4, 6×1]
       = [24,   12,   8,   6]  ✓
```

**Complexity:** Time O(n) | Space O(1) excluding output array

---

## Q201. 3Sum

**Problem:** `[-1,0,1,2,-1,-4]` → `[[-1,-1,2],[-1,0,1]]` (no duplicate triplets)

💡 **Strategy:** "Sort first. Fix one number, then use Two Pointers on the rest to find pairs summing to its negative."

### 🔴 Brute Force — O(n³)
```java
public List<List<Integer>> threeSum(int[] nums) {
    Set<List<Integer>> result = new HashSet<>();
    int n = nums.length;
    for (int i = 0; i < n - 2; i++)
        for (int j = i + 1; j < n - 1; j++)
            for (int k = j + 1; k < n; k++)
                if (nums[i] + nums[j] + nums[k] == 0) {
                    List<Integer> triplet = Arrays.asList(nums[i], nums[j], nums[k]);
                    Collections.sort(triplet);
                    result.add(triplet);
                }
    return new ArrayList<>(result);
}
```

### 🟢 Optimal — Sort + Two Pointers O(n²)
```java
public List<List<Integer>> threeSum(int[] nums) {
    Arrays.sort(nums);                        // Sort first!
    List<List<Integer>> result = new ArrayList<>();

    for (int i = 0; i < nums.length - 2; i++) {
        if (i > 0 && nums[i] == nums[i - 1]) continue;  // Skip duplicate 'i'
        if (nums[i] > 0) break;                          // All remaining nums[i] > 0 → no triplet sums to 0

        int lo = i + 1;
        int hi = nums.length - 1;

        while (lo < hi) {
            int sum = nums[i] + nums[lo] + nums[hi];
            if (sum == 0) {
                result.add(Arrays.asList(nums[i], nums[lo], nums[hi]));
                while (lo < hi && nums[lo] == nums[lo + 1]) lo++;  // Skip duplicates
                while (lo < hi && nums[hi] == nums[hi - 1]) hi--;  // Skip duplicates
                lo++;
                hi--;
            } else if (sum < 0) {
                lo++;   // Need a larger number
            } else {
                hi--;   // Need a smaller number
            }
        }
    }
    return result;
}
```

**Trace:** `[-4,-1,-1,0,1,2]` (sorted)
```
i=0 (-4): lo=1 hi=5 → -4+-1+2=-3 → lo++
         lo=2 hi=5 → -4+-1+2=-3 → lo++
         lo=3 hi=5 → -4+0+2=-2  → lo++
         lo=4 hi=5 → -4+1+2=-1  → lo++  (lo=hi, exit)
i=1 (-1): lo=2 hi=5 → -1+-1+2=0 → add [-1,-1,2] ✓, lo→3, hi→4
         lo=3 hi=4 → -1+0+1=0   → add [-1,0,1]  ✓, exit
i=2 (-1): nums[2]==nums[1] → skip
```

**Complexity:** Time O(n²) | Space O(1)

---

## Q202. Longest Palindromic Substring

**Problem:** `"babad"` → `"bab"` | `"cbbd"` → `"bb"`

💡 **Strategy:** "Every palindrome expands from a center. Try each character (and each gap) as center and expand outward as long as left==right."

### 🔴 Brute Force — O(n³)
```java
public String longestPalindrome(String s) {
    String longest = "";
    for (int i = 0; i < s.length(); i++) {
        for (int j = i; j <= s.length(); j++) {
            String sub = s.substring(i, j);
            if (isPalindrome(sub) && sub.length() > longest.length()) {
                longest = sub;
            }
        }
    }
    return longest;
}
private boolean isPalindrome(String s) {
    int lo = 0, hi = s.length() - 1;
    while (lo < hi) if (s.charAt(lo++) != s.charAt(hi--)) return false;
    return true;
}
```

### 🟢 Optimal — Expand Around Center O(n²)
```java
private int start = 0, maxLen = 1;

public String longestPalindrome(String s) {
    if (s.length() < 2) return s;

    for (int i = 0; i < s.length() - 1; i++) {
        expand(s, i, i);       // Odd length: center is one char  "aba"
        expand(s, i, i + 1);   // Even length: center is a gap    "abba"
    }

    return s.substring(start, start + maxLen);
}

private void expand(String s, int lo, int hi) {
    while (lo >= 0 && hi < s.length() && s.charAt(lo) == s.charAt(hi)) {
        int currentLen = hi - lo + 1;
        if (currentLen > maxLen) {
            maxLen = currentLen;
            start  = lo;        // Remember start position
        }
        lo--;
        hi++;
    }
}
```

**Trace:** `"babad"`
```
i=0, center='b': expand → "b" (len 1)
i=1, center='a': expand → "bab" (lo=0,hi=2, len 3) ✓ → try expand "aba"? lo=-1 stop
i=1, gap='a'+'b': 'a'!='b' → no expansion
i=2, center='b': expand → "aba" (lo=1,hi=3, len 3) same length, keep "bab"
...
```

**Complexity:** Time O(n²) | Space O(1)

---

## Q203. Sort Colors (Dutch National Flag)

**Problem:** `[2,0,2,1,1,0]` → `[0,0,1,1,2,2]` in-place, O(n), one pass.

💡 **Strategy:** "Three pointers: `lo` is the 0-section end, `hi` is the 2-section start, `mid` scans. Swap into place."

### 🔴 Brute Force — Counting Sort O(n)
```java
public void sortColors(int[] nums) {
    int count0 = 0, count1 = 0, count2 = 0;
    for (int n : nums) {
        if (n == 0) count0++;
        else if (n == 1) count1++;
        else count2++;
    }
    int i = 0;
    while (count0-- > 0) nums[i++] = 0;
    while (count1-- > 0) nums[i++] = 1;
    while (count2-- > 0) nums[i++] = 2;
}
// 2 passes — valid but doesn't use the optimal 1-pass approach
```

### 🟢 Optimal — Dutch National Flag (1 pass)
```java
public void sortColors(int[] nums) {
    int lo  = 0;                   // Next position for 0
    int mid = 0;                   // Current element being examined
    int hi  = nums.length - 1;     // Next position for 2

    while (mid <= hi) {
        if (nums[mid] == 0) {
            swap(nums, lo++, mid++);   // 0 goes to lo section, advance both
        } else if (nums[mid] == 1) {
            mid++;                     // 1 is already in correct region
        } else {                       // nums[mid] == 2
            swap(nums, mid, hi--);     // 2 goes to hi section, DON'T advance mid
                                       // (swapped value from hi not yet checked)
        }
    }
}

private void swap(int[] a, int i, int j) {
    int tmp = a[i]; a[i] = a[j]; a[j] = tmp;
}
```

**Trace:** `[2,0,2,1,1,0]`
```
lo=0, mid=0, hi=5
mid=2: swap(0,5) → [0,0,2,1,1,2], hi=4
mid=0: swap(0,0) → [0,0,2,1,1,2], lo=1, mid=1
mid=0: swap(1,1) → [0,0,2,1,1,2], lo=2, mid=2
mid=2: swap(2,4) → [0,0,1,1,2,2], hi=3
mid=1: mid=3
mid=1: mid=4  (mid > hi=3, exit)
Result: [0,0,1,1,2,2] ✓
```

**Complexity:** Time O(n) | Space O(1) — single pass

---

## Q204. Spiral Matrix

**Problem:** `[[1,2,3],[4,5,6],[7,8,9]]` → `[1,2,3,6,9,8,7,4,5]`

💡 **Strategy:** "Shrink boundaries: traverse right (top row), down (right col), left (bottom row), up (left col). After each direction, shrink that boundary inward."

### 🟢 Solution — Boundary Shrink O(m×n)
```java
public List<Integer> spiralOrder(int[][] matrix) {
    List<Integer> result = new ArrayList<>();
    if (matrix.length == 0) return result;

    int top    = 0;
    int bottom = matrix.length - 1;
    int left   = 0;
    int right  = matrix[0].length - 1;

    while (top <= bottom && left <= right) {
        // → Traverse right along top row
        for (int col = left; col <= right; col++)
            result.add(matrix[top][col]);
        top++;

        // ↓ Traverse down along right column
        for (int row = top; row <= bottom; row++)
            result.add(matrix[row][right]);
        right--;

        // ← Traverse left along bottom row (if row still exists)
        if (top <= bottom) {
            for (int col = right; col >= left; col--)
                result.add(matrix[bottom][col]);
            bottom--;
        }

        // ↑ Traverse up along left column (if column still exists)
        if (left <= right) {
            for (int row = bottom; row >= top; row--)
                result.add(matrix[row][left]);
            left++;
        }
    }

    return result;
}
```

**Trace:** `[[1,2,3],[4,5,6],[7,8,9]]`
```
→ top row:     1,2,3     top=1
↓ right col:   6,9       right=1
← bottom row:  8,7       bottom=1
↑ left col:    4         left=1
→ top row:     5         (top=1, bottom=1, left=1, right=1)
Result: [1,2,3,6,9,8,7,4,5] ✓
```

**Complexity:** Time O(m×n) | Space O(1) excluding output

---

## Q205. Container With Most Water

**Problem:** `[1,8,6,2,5,4,8,3,7]` → `49` (lines at index 1 and 8, height min(8,7)=7, width=7, area=49)

💡 **Strategy:** "Two pointers converge inward. Always move the SHORTER side — the taller side can't do better with the shorter one, so try to find a taller partner."

### 🔴 Brute Force — O(n²)
```java
public int maxArea(int[] height) {
    int maxArea = 0;
    for (int i = 0; i < height.length; i++) {
        for (int j = i + 1; j < height.length; j++) {
            int area = Math.min(height[i], height[j]) * (j - i);
            maxArea = Math.max(maxArea, area);
        }
    }
    return maxArea;
}
```

### 🟢 Optimal — Two Pointers O(n)
```java
public int maxArea(int[] height) {
    int lo  = 0;
    int hi  = height.length - 1;
    int max = 0;

    while (lo < hi) {
        int h    = Math.min(height[lo], height[hi]);   // Shorter wall limits water
        int area = h * (hi - lo);                       // Width × height
        max = Math.max(max, area);

        // Move the shorter pointer inward
        // Why? Moving the taller one can only decrease or maintain width,
        // and the height is still limited by the shorter one — no benefit
        if (height[lo] < height[hi]) {
            lo++;
        } else {
            hi--;
        }
    }

    return max;
}
```

**Trace:** `[1,8,6,2,5,4,8,3,7]`
```
lo=0(h=1), hi=8(h=7): area=min(1,7)*8=8,   move lo (shorter)
lo=1(h=8), hi=8(h=7): area=min(8,7)*7=49,  move hi (shorter) ← max!
lo=1(h=8), hi=7(h=3): area=min(8,3)*6=18,  move hi
lo=1(h=8), hi=6(h=8): area=min(8,8)*5=40,  move hi (or lo, equal)
...
max = 49 ✓
```

**Why move shorter side?** Keeping the taller side and moving it inward: width shrinks, height can only stay same or decrease (still limited by shorter). Keeping the shorter side has no hope of beating current area. Moving shorter side gives a chance of finding a taller partner.

**Complexity:** Time O(n) | Space O(1)

---

## Master Cheat Sheet — Patterns & Strategies

| Q | Problem | Pattern | One-Liner Memory Tip |
|---|---|---|---|
| **Q190** | Valid Parentheses | Stack | Push expected closer; pop and compare on close |
| **Q191** | Buy & Sell Stock | Greedy | Track `minPrice` seen; profit = today − min |
| **Q192** | Longest No-Repeat Substr | Sliding Window + Map | Window shrinks left when duplicate enters |
| **Q193** | Max Subarray (Kadane) | DP (running max) | `curr = max(num, curr+num)` — restart if negative |
| **Q194** | Subarray Sum = K | Prefix Sum + HashMap | `count += freq[currSum - k]` |
| **Q195** | Contains Duplicate | HashSet | `!set.add(n)` returns true on duplicate |
| **Q196** | Missing Number | Math / XOR | `n*(n+1)/2 - sum` or XOR all indices with values |
| **Q197** | Intersection | HashSet | Load set1, walk nums2 looking for hits |
| **Q198** | Move Zeroes | Two Pointers | `left` pointer collects non-zeros; fill zeros at end |
| **Q199** | Rotate Array | Reverse Trick | Reverse all → reverse [0..k-1] → reverse [k..n-1] |
| **Q200** | Product Except Self | Prefix × Suffix | Left pass builds prefix products; right pass multiplies suffix |
| **Q201** | 3Sum | Sort + Two Pointers | Fix `nums[i]`; two-pointer on remaining; skip duplicates |
| **Q202** | Longest Palindrome | Expand Around Center | Expand from each center (odd + even). Track `start` + `maxLen` |
| **Q203** | Sort Colors | Dutch National Flag | lo/mid/hi; 0→swap lo++mid++, 1→mid++, 2→swap hi-- |
| **Q204** | Spiral Matrix | Boundary Shrink | `top, bottom, left, right` walls; shrink after each side |
| **Q205** | Container With Most Water | Two Pointers | Move the SHORTER wall inward — taller wall can't help without taller partner |

### Universal Patterns Reference

```
Two Pointers     → Sorted array or finding pairs/triplets (3Sum, Container)
Sliding Window   → Substring/subarray problems with constraints (No-Repeat)
Prefix Sum+Map   → Subarray sum equals target (Subarray Sum = K)
HashSet          → Duplicate detection, intersection
Stack            → Matching brackets, next greater element
Greedy           → Local optimal = global optimal (Stock, Kadane)
Divide by 2      → Binary search on sorted/rotated arrays
Reverse trick    → Rotate array in-place (O(1) space)
Expand center    → Palindrome problems
Boundary shrink  → Matrix traversal (Spiral)
Dutch 3-pointer  → Partition into 3 groups (Sort Colors)
```

---

**End of Sections 17 & 18**

---

## Additional Deep-Dive (Q190-Q205)

### Pattern Selection Heuristics

- Sliding window: contiguous subarray/substring with moving boundary constraints.
- Two pointers: sorted arrays or inward scans with pair/triplet goals.
- Hash map/set: lookup-heavy problems where $O(1)$ membership changes complexity class.
- Dynamic programming: overlapping subproblems + optimal substructure.

### Real Project Usage

- Even in backend teams, DSA patterns appear in log processing, recommendation scoring, fraud checks, and scheduling heuristics.
- Senior expectation is not just coding the solution, but selecting the correct pattern quickly and defending why alternatives are weaker.
