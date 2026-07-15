# CODING PATTERNS FOR INTERVIEWS
## A Book-Level Guide to Recognizing, Understanding, and Applying Patterns

---

> **How to use this document.** This is not a list of templates to memorize. It is a guide to *thinking* like someone who can solve an unfamiliar problem in 25 minutes. For every pattern you will find: **when** it triggers, **why** it works, **how** to build the solution step by step, the complexity reasoning, and the mistakes that turn clean code into buggy code. Read for understanding first; the code is the proof, not the lesson.

---

## TABLE OF CONTENTS

1. [The Interview Problem-Solving Framework](#1-the-interview-problem-solving-framework)
2. [How to Recognize a Pattern (the decision flow)](#2-how-to-recognize-a-pattern)
3. [Arrays & Strings](#3-arrays--strings)
   - 3.1 Sliding Window
   - 3.2 Two Pointers
   - 3.3 Binary Search
   - 3.4 Prefix Sum & Hash Map
   - 3.5 Monotonic Stack
   - 3.6 Intervals
   - 3.7 Backtracking (Subsets, Permutations, Combinations)
4. [Linked Lists](#4-linked-lists)
   - 4.1 Fast & Slow Pointers
   - 4.2 Reversal & Dummy-Node Patterns
5. [Trees](#5-trees)
   - 5.1 Tree DFS (recursion)
   - 5.2 Tree BFS (level order)
   - 5.3 BST properties
6. [Graphs](#6-graphs)
   - 6.1 Graph DFS / BFS
   - 6.2 Union Find
   - 6.3 Topological Sort
   - 6.4 Shortest Path
7. [Dynamic Programming](#7-dynamic-programming)
   - 7.1 When DP applies
   - 7.2 1D DP
   - 7.3 Grid / 2D DP
   - 7.4 String DP
   - 7.5 Interval & Knapsack DP
8. [Heap, Divide & Conquer, Trie, Bits](#8-advanced-data-structures--tricks)
   - 8.1 Heap / Priority Queue
   - 8.2 Quick Select & Divide & Conquer
   - 8.3 Trie
   - 8.4 Bit Manipulation
9. [Anti-Patterns & Confusion Traps](#9-anti-patterns--confusion-traps)
10. [Practice Roadmap & Anki Cards](#10-practice-roadmap)

---

## 1. THE INTERVIEW PROBLEM-SOLVING FRAMEWORK

The single biggest cause of failing a coding interview is not bad algorithms — it is not having a *process*. Interviewers grade the process. Use the same six steps every single time, even on easy problems. It keeps you calm and gives the interviewer a clear story to follow.

### Step 1 — Clarify (1–2 min)
Never start coding before you understand the problem. Ask, in this order:
- **Input size & type** — "How large can `n` be?" This single question decides whether O(n²) is acceptable or fatal.
- **Constraints / edge cases** — "Can the array be empty? Can there be negative numbers? Duplicates? Is it sorted?"
- **Output format** — "Return indices or values? All solutions or one? Is order important?"
- **Examples** — Walk one example input to output to confirm you understood.

> **Why this matters:** Clarifying signals seniority. A junior reaches for code; a senior reaches for the problem statement. It also often reveals the intended pattern (e.g., "sorted" → binary search or two pointers).

### Step 2 — Brute force (2 min)
State the naive solution and its complexity out loud. This guarantees you always have *a* working answer, buys thinking time, and is a baseline the interviewer respects.

> "I can check every pair, which is O(n²). Let me see if we can do better."

### Step 3 — Optimize (3–5 min)
This is where pattern recognition lives (see Section 2). Brainstorm 1–3 approaches, compare time/space trade-offs, and *pick one with justification*. Say which you'll implement and why.

### Step 4 — Walk through an example (2 min)
Trace your chosen approach on a tiny input line by line. This catches off-by-one errors before they reach code and shows the interviewer your logic.

### Step 5 — Implement (8–12 min)
Write clean, readable code. Name variables by meaning (`left`, `right`, `window_sum`), not `i`, `j`, `tmp`. Handle edge cases up front.

### Step 6 — Test & complexity (2 min)
Run your code on 2–3 cases including an edge case. State time and space complexity and *why*.

```
CLARIFY → BRUTE FORCE → OPTIMIZE → WALK EXAMPLE → IMPLEMENT → TEST
```

---

## 2. HOW TO RECOGNIZE A PATTERN

When you read a problem, run this decision flow **top to bottom**. Stop at the first match. The keywords in quotes are the literal trigger phrases.

```
START
 │
 ├─ "contiguous subarray / substring" with a condition (sum, unique chars, length)
 │     → SLIDING WINDOW (3.1)
 │
 ├─ "sorted array" + find pair / triplet / target
 │     → TWO POINTERS (3.2)
 │
 ├─ "sorted" + find a value / first occurrence / answer is a number you can guess-and-check
 │     → BINARY SEARCH (3.3)
 │
 ├─ "subarray sum equals K" / "how many subarrays" / "range sum"
 │     → PREFIX SUM & HASH MAP (3.4)
 │
 ├─ "next greater / next smaller" / "largest rectangle" / "stock span"
 │     → MONOTONIC STACK (3.5)
 │
 ├─ "merge overlapping intervals" / "meeting rooms" / "insert and merge"
 │     → INTERVALS (3.6)
 │
 ├─ "all combinations / permutations / subsets" / "find all paths"
 │     → BACKTRACKING (3.7)
 │
 ├─ "linked list" + cycle / middle / palindrome
 │     → FAST & SLOW POINTERS (4.1)
 │
 ├─ "linked list" + reverse / merge / group
 │     → REVERSAL & DUMMY NODE (4.2)
 │
 ├─ "tree" + level-by-level / shortest path in tree
 │     → TREE BFS (5.2)
 │
 ├─ "tree" + path / ancestry / in-order
 │     → TREE DFS (5.1)
 │
 ├─ "connected components" / "number of islands" / dynamic connectivity
 │     → UNION FIND (6.2) or GRAPH DFS/BFS (6.1)
 │
 ├─ "dependencies / prerequisites / course schedule / build order"
 │     → TOPOLOGICAL SORT (6.3)
 │
 ├─ "shortest path" + weighted graph → DIJKSTRA (6.4); unweighted → BFS (6.1)
 │
 ├─ "optimal / maximum / minimum ... ways to ... " + repeated subproblems
 │     → DYNAMIC PROGRAMMING (7)
 │
 ├─ "kth largest / smallest" / "top K" / "median of stream"
 │     → HEAP (8.1) or QUICK SELECT (8.2)
 │
 ├─ "prefix matching / autocomplete / words sharing prefix"
 │     → TRIE (8.3)
 │
 └─ "bitwise / single number / power of two / XOR"
       → BIT MANIPULATION (8.4)
```

**Key insight:** The *data structure* in the problem is your biggest hint. Sorted → two pointers / binary search. Linked list → pointers. Tree → recursion. Graph → BFS/DFS/UF. Subarray → window / prefix / hash map.

---

## 3. ARRAYS & STRINGS

### 3.1 Sliding Window

**Trigger:** "contiguous subarray/substring" with a constraint — maximum sum of size `k`, smallest subarray ≥ `target`, longest substring without repeating characters, minimum window containing a set of characters. *Contiguous* is the keyword; if it said "subsequence" this pattern would not apply.

**Core intuition:** A window `[left, right]` slides across the array. Instead of recomputing the window's property from scratch each step, you **add the entering element and remove the leaving element** — O(1) work per slide instead of O(k). That turns an O(n·k) naive scan into O(n).

**Why brute force fails:** Checking every window of every start is O(n²) or O(n·k). The window pattern does it in one pass.

**The recipe:**
1. Initialize `left = 0` and a `window_state` (sum, or a frequency map for characters).
2. Expand `right` from 0 to n−1, updating `window_state` with `nums[right]`.
3. **Fixed window** (size known): once `right - left + 1 == k`, record answer and move both pointers.
4. **Variable window** (condition-based): while the window *violates* the condition, shrink from `left` (remove `nums[left]`, `left += 1`). After each valid window, record the best answer.
5. Return the best recorded answer.

**Reference implementation (variable window — smallest subarray ≥ target):**

```python
def minSubArrayLen(target, nums):
    left = 0
    current_sum = 0
    min_len = float('inf')          # "no answer found yet"
    for right in range(len(nums)):
        current_sum += nums[right]   # expand: include the new element
        # shrink while the window already satisfies the goal
        while current_sum >= target:
            min_len = min(min_len, right - left + 1)
            current_sum -= nums[left]  # remove the element leaving the window
            left += 1
    return min_len if min_len != float('inf') else 0
```

**Fixed window (maximum sum of `k` consecutive elements):**

```python
def maxSumSubarray(nums, k):
    window_sum = sum(nums[:k])      # compute the first window once
    max_sum = window_sum
    for i in range(k, len(nums)):
        window_sum += nums[i] - nums[i - k]   # add new, drop old — O(1)
        max_sum = max(max_sum, window_sum)
    return max_sum
```

**Complexity:** Time O(n) — each element enters and leaves the window at most once. Space O(1) for sum problems, O(1) to O(k) for character-frequency maps (bounded by alphabet size).

**Variants:**
- Fixed-size window (max/min sum, average).
- Condition window (subarray sum ≥ target, at most K distinct).
- Window with a character map (longest substring without repeats, minimum window substring).

**Classic problems & the one insight:**
| Problem | Insight |
|---------|---------|
| Longest Substring Without Repeating | Map char→last index; when a repeat appears, jump `left` to `seen[char] + 1` (not `left += 1`). |
| Minimum Window Substring | Two maps (need vs. have); track a `formed` counter so you shrink only when fully satisfied. |
| Longest Repeating Character Replacement | Window is valid if `window_len - max_freq <= k`. |

**Common mistakes:**
- *Shrinking one step at a time when you can jump.* For longest-substring-without-repeat, set `left = max(left, seen[char] + 1)`; don't loop `left += 1`.
- *Off-by-one in length:* length is `right - left + 1`, not `right - left`.
- *Forgetting the `float('inf')` sentinel* and returning it when no answer exists.

---

### 3.2 Two Pointers

**Trigger:** A **sorted** array (or one you can sort) and you need pairs/triplets summing to a target, or in-place deduplication/reversal, or "container with most water."

**Core intuition:** With a sorted array, the sum of the two ends tells you which direction to move. If the sum is too small, move the left pointer right (bigger values); if too big, move the right pointer left. You explore the search space in O(n) instead of checking all O(n²) pairs.

**Why it requires sorting:** The "move the smaller/larger end" decision is only valid because order is known. If the array isn't sorted, sort it first (O(n log n)) — still far better than O(n²).

**The recipe (Two Sum on sorted array):**
1. `left = 0`, `right = n - 1`.
2. While `left < right`:
   - `s = nums[left] + nums[right]`.
   - If `s == target` → return.
   - If `s < target` → `left += 1` (need bigger).
   - Else → `right -= 1` (need smaller).

```python
def twoSumSorted(nums, target):
    left, right = 0, len(nums) - 1
    while left < right:
        s = nums[left] + nums[right]
        if s == target:
            return [left, right]
        elif s < target:
            left += 1
        else:
            right -= 1
    return []
```

**Container With Most Water (greedy two-pointer — the subtle one):**

```python
def maxArea(height):
    left, right = 0, len(height) - 1
    max_area = 0
    while left < right:
        area = min(height[left], height[right]) * (right - left)
        max_area = max(max_area, area)
        # Move the SHORTER line inward. Moving the taller one can never
        # increase area (width shrinks, height capped by the shorter).
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1
    return max_area
```

**In-place deduplication (read/write pointers):**

```python
def removeDuplicates(nums):
    if not nums:
        return 0
    write = 1                      # next position to write a kept element
    for read in range(1, len(nums)):
        if nums[read] != nums[write - 1]:   # compare to last kept, not nums[read-1]
            nums[write] = nums[read]
            write += 1
    return write                  # new length
```

**Complexity:** Time O(n) after sorting; O(n) for in-place transforms. Space O(1) (ignoring sort stack).

**Variant — 3Sum:** Sort, then for each `i`, run two pointers on `i+1..n-1`. Skip duplicates of `i`, `left`, and `right` to avoid duplicate triplets.

**Common mistakes:**
- Using two pointers on an **unsorted** array (wrong — the logic depends on order).
- In `removeDuplicates`, comparing `nums[read]` to `nums[read-1]` instead of `nums[write-1]` — they differ once you've overwritten.
- Infinite loops in `3Sum` from not advancing past duplicates correctly.

---

### 3.3 Binary Search

**Trigger:** A **sorted** array and you need to find a value, its first/last occurrence, or a value in a rotated sorted array. Also — and this is the高级 use — when the **answer is a number in a range and you can test "is this answer good enough?"** (e.g., "find the smallest capacity that works").

**Core intuition:** Each comparison throws away half the search space. That's why it's O(log n). The subtle, powerful version — *binary search on the answer* — works whenever the function "feeds a capacity, returns feasible?" is **monotonic** (all answers ≥ X work, all < X fail). You binary-search over the answer space instead of an index space.

**The recipe (standard find target):**

```python
def binarySearch(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:                 # <= because [left,right] is inclusive
        mid = left + (right - left) // 2 # avoid overflow in other languages
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1               # mid is too small, discard it
        else:
            right = mid - 1
    return -1
```

**Lower bound (first index where `nums[i] >= target`):** This is the template for "first occurrence" and for answer-space search.

```python
def lowerBound(nums, target):
    left, right = 0, len(nums)          # right is PAST the end (exclusive)
    while left < right:                  # < because [left, right) has no mid==right
        mid = left + (right - left) // 2
        if nums[mid] < target:
            left = mid + 1
        else:
            right = mid                  # mid might be the answer, keep it
    return left
```

**Search in Rotated Sorted Array:** The pivot splits the array into two sorted halves; exactly one of `nums[left..mid]` or `nums[mid..right]` is fully sorted.

```python
def searchRotated(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = (left + right) // 2
        if nums[mid] == target:
            return mid
        if nums[left] <= nums[mid]:                 # left half is sorted
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        else:                                        # right half is sorted
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1
    return -1
```

**Complexity:** Time O(log n) standard; O(log(max−min)) for answer-space search. Space O(1).

**Common mistakes:**
- Mixing up `left <= right` (inclusive) vs `left < right` (exclusive) — the bound style must match how you update `right`.
- Integer overflow in `mid = (left + right) // 2` for huge arrays in C/Java — use `left + (right-left)//2`.
- Forgetting binary search applies to **any monotonic predicate**, not just sorted arrays. This is the unlock for "minimize the maximum" problems (ship packages, split array largest sum, capacity to ship in D days).

---

### 3.4 Prefix Sum & Hash Map

**Trigger:** "subarray sum equals K," "number of subarrays with sum X," "range sum queries," or "maximum subarray sum" (Kadane's). The shared idea: turn a subarray question into a **running total + lookup** question.

**Core intuition (subarray sum):** Let `P[i]` = sum of `nums[0..i]`. Then sum of `nums[l..r]` = `P[r] - P[l-1]`. So a subarray summing to `K` exists iff some earlier prefix `P[l-1]` equals `P[r] - K`. As you walk once, store every prefix sum you've seen in a hash map and check for `current - K`. O(n) instead of O(n²).

```python
def subarraySum(nums, k):
    count = 0
    prefix = 0
    freq = {0: 1}          # crucial: a prefix of 0 has occurred before we start
    for num in nums:
        prefix += num
        if prefix - k in freq:        # is there an earlier prefix we can subtract?
            count += freq[prefix - k]
        freq[prefix] = freq.get(prefix, 0) + 1
    return count
```

**Kadane's Algorithm (maximum subarray sum):** At each position, decide: extend the previous subarray, or start fresh here. Track the best seen.

```python
def maxSubArray(nums):
    best = cur = nums[0]       # 'cur' = max subarray ENDING at current index
    for x in nums[1:]:
        cur = max(x, cur + x)  # start new, or extend
        best = max(best, cur)
    return best
```

**Longest Substring Without Repeating (prefix idea + window):** covered in 3.1; uses a char→index map.

**Complexity:** Time O(n), Space O(n) for the frequency map.

**Common mistakes:**
- Omitting the `{0: 1}` seed, which makes subarrays starting at index 0 invisible.
- In Kadane's, initializing `best = 0` — wrong when all numbers are negative (answer is the largest negative). Initialize with `nums[0]`.

---

### 3.5 Monotonic Stack

**Trigger:** "next greater element," "next smaller element," "daily temperatures," "largest rectangle in histogram," "stock span." The signature: you need, for each element, the **nearest element to one side that is bigger/smaller.**

**Core intuition:** Maintain a stack of indices whose values are *monotonically* increasing (or decreasing). When a new element breaks the monotonicity, the elements it pops have found their "next greater" answer — it's the current element. Each element is pushed and popped at most once → O(n).

```python
def nextGreaterElement(nums):
    stack = []
    result = [-1] * len(nums)
    for i, num in enumerate(nums):
        # pop everything smaller than num: num is their "next greater"
        while stack and nums[stack[-1]] < num:
            idx = stack.pop()
            result[idx] = num
        stack.append(i)        # push; stack stays increasing bottom→top
    return result
```

**Largest Rectangle in Histogram (the hard one):** For each bar, the largest rectangle using it as height extends left until a shorter bar and right until a shorter bar. The monotonic stack finds those boundaries. Append a `0` at the end to flush all remaining bars.

```python
def largestRectangleArea(heights):
    stack = []
    max_area = 0
    for i, h in enumerate(heights + [0]):      # 0 forces every bar to pop
        while stack and heights[stack[-1]] > h:
            height = heights[stack.pop()]
            # width = distance from previous smaller bar to current
            width = i if not stack else i - stack[-1] - 1
            max_area = max(max_area, height * width)
        stack.append(i)
    return max_area
```

**Complexity:** Time O(n) — each index pushed and popped once. Space O(n).

**Common mistakes:**
- Building an *increasing* stack when the problem needs "next smaller" (use `>` vs `<` carefully).
- Forgetting the trailing sentinel (`+ [0]` or a final flush loop) so bars remaining in the stack never get processed.

---

### 3.6 Intervals

**Trigger:** "merge overlapping intervals," "insert and merge," "meeting rooms (how many concurrent?)," "non-overlapping intervals to remove." Nearly always: **sort by start time first**, then either merge or count.

**Core intuition:** Once intervals are sorted by start, an interval can only overlap the *current* merged interval (because everything earlier starts even earlier). So you keep one "current" interval and extend its end if the next overlaps; otherwise emit it and start a new one.

```python
def mergeIntervals(intervals):
    intervals.sort(key=lambda x: x[0])     # sort by start
    merged = [intervals[0]]
    for start, end in intervals[1:]:
        last_start, last_end = merged[-1]
        if start <= last_end:              # overlap (note: <= merges touching)
            merged[-1] = (last_start, max(last_end, end))
        else:
            merged.append((start, end))
    return merged
```

**Variant — Meeting Rooms II (minimum rooms):** Count maximum overlap. Two approaches: (a) sort starts and ends separately and sweep; (b) a min-heap of end times — when a meeting starts after the earliest end, reuse that room.

**Complexity:** Time O(n log n) for the sort, O(n) after. Space O(n) or O(1) excluding output.

**Common mistakes:**
- Forgetting to sort first — the merge logic is wrong on unsorted input.
- Using `<` vs `<=` for "overlap": touching intervals (`[1,2]` and `[2,3]`) overlap only if the problem says so; pick deliberately.

---

### 3.7 Backtracking (Subsets, Permutations, Combinations)

**Trigger:** "find *all* valid combinations / permutations / subsets," "all paths," "generate parentheses," "word search." You need to explore a search tree and collect every answer.

**Core intuition:** Build candidates one choice at a time. After exploring a branch, **undo the last choice** (backtrack) so the same partial state can be reused for the next branch. This is DFS over a state space with pruning.

**The recipe:**
1. A `result` list, a `path` (current choice), and a `start` index (to avoid revisiting earlier elements in combinations/subsets).
2. At each call: if `path` is a complete valid answer, add a *copy* to `result`.
3. Loop over candidates from `start` (or 0 for permutations): choose, recurse, **undo**.
4. Prune early when the partial path can never become valid.

```python
def subsets(nums):
    result = []
    path = []
    def backtrack(start):
        result.append(path[:])            # every prefix is a valid subset
        for i in range(start, len(nums)):
            path.append(nums[i])
            backtrack(i + 1)              # move forward to avoid duplicates
            path.pop()                    # UNDO — backtrack
    backtrack(0)
    return result
```

**Permutations** differs only in that every unused element is a candidate each level (track `used`), not just those after `start`.

**Complexity:** Exponential in general (there are 2ⁿ subsets, n! permutations) — unavoidable because the output itself is that large. Space O(n) for the recursion depth.

**Common mistakes:**
- Appending `path` instead of `path[:]` (a copy) — every result ends up pointing to the same mutated list.
- Forgetting `path.pop()` — the backtrack step; without it you never explore siblings.
- Not pruning, leading to TLE on large inputs (e.g., combination sum should stop when `target < 0`).

---

## 4. LINKED LISTS

### 4.1 Fast & Slow Pointers (Floyd's Tortoise & Hare)

**Trigger:** "detect a cycle," "find the start of a cycle," "middle of the list," "palindrome linked list," "find duplicate number" (array form).

**Core intuition:** Two pointers move at different speeds. If there's a cycle, the fast pointer eventually laps the slow one (they meet). If there's no cycle, fast hits the end. This detects a cycle in O(n) time and O(1) space — versus hashing every node (O(n) space).

**Cycle detection:**

```python
def hasCycle(head):
    slow = fast = head
    while fast and fast.next:       # fast moves 2, needs two nodes ahead
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            return True
    return False
```

**Find the cycle start (the elegant part):** After they meet, reset one pointer to `head`. Move both one step at a time; they meet at the cycle's entry. *Why it works:* the distance from head to cycle-start equals the distance from the meeting point to cycle-start (modulo the cycle length). This is worth being able to explain.

```python
def detectCycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            slow = head
            while slow != fast:
                slow = slow.next
                fast = fast.next
            return slow
    return None
```

**Middle of the list:** When `fast` reaches the end, `slow` is at the middle. With `fast and fast.next` test, `slow` lands on the *second* middle for even lengths (useful for later splitting).

**Complexity:** O(n) time, O(1) space.

**Common mistakes:**
- Testing `while fast:` instead of `while fast and fast.next:` → `fast.next.next` crashes on a None.
- Forgetting the second loop starts with `slow = head` (not `slow = slow.next`) in `detectCycle`.

---

### 4.2 Reversal & Dummy-Node Patterns

**Trigger:** "reverse a linked list (full or partial)," "reverse nodes in k-groups," "merge two sorted lists," "remove nth node from end," "partition list." Themes: **iterative pointer rewiring** and **dummy nodes** to avoid head-edge-case bugs.

**Iterative reversal (3 pointers):**

```python
def reverseList(head):
    prev = None
    curr = head
    while curr:
        nxt = curr.next      # 1. save next
        curr.next = prev     # 2. flip pointer
        prev = curr          # 3. advance prev
        curr = nxt           # 4. advance curr
    return prev              # new head
```

**Dummy node (remove nth node from end):** A dummy before `head` means removing the real head uses the same code as removing any node.

```python
def removeNthFromEnd(head, n):
    dummy = ListNode(0, head)
    fast = slow = dummy
    for _ in range(n + 1):       # gap of n+1 between fast and slow
        fast = fast.next
    while fast:
        fast = fast.next
        slow = slow.next
    slow.next = slow.next.next   # skip the target
    return dummy.next
```

**Complexity:** O(n) time, O(1) space (dummy is O(1)).

**Common mistakes:**
- Losing the next pointer before rewiring in reversal (always save `nxt` first).
- Not using a dummy → special-casing head removal and off-by-one bugs.

---

## 5. TREES

### 5.1 Tree DFS (recursion)

**Trigger:** "all root-to-leaf paths," "path sum," "lowest common ancestor," "is this tree valid," "in-order traversal." Trees are recursive by nature, so most tree problems are a recursive function returning/accumulating something per subtree.

**Core intuition:** Solve the problem for a node by solving it for `left` and `right`, then combine. Pick the traversal order (pre/in/post) by *when* you need the node's value relative to its children.

**Path Sum (does any root→leaf equal target):**

```python
def hasPathSum(root, target):
    if not root:
        return False
    if not root.left and not root.right:     # leaf
        return root.val == target
    return (hasPathSum(root.left, target - root.val) or
            hasPathSum(root.right, target - root.val))
```

**Lowest Common Ancestor (the return-value trick):** Return the node if it's `p`/`q` or if one child found `p` and the other found `q`.

```python
def lowestCommonAncestor(root, p, q):
    if not root or root == p or root == q:
        return root
    left = lowestCommonAncestor(root.left, p, q)
    right = lowestCommonAncestor(root.right, p, q)
    if left and right:        # p and q split across children → this is LCA
        return root
    return left or right      # otherwise pass up whichever found something
```

**Complexity:** O(n) time (visit each node once), O(h) space for the recursion stack, where h = tree height (O(n) worst case, O(log n) balanced).

**Common mistakes:**
- Base case only `if not root` and forgetting leaf handling for path problems.
- Stack overflow on a degenerate (linked-list) tree — mention iterative/stack alternative if n is huge.

---

### 5.2 Tree BFS (level order)

**Trigger:** "return nodes level by level," "leftmost/rightmost at each level," "minimum depth," "shortest path in a tree/unedited graph." BFS finds the *shortest* path in unweighted structures and visits level-by-level.

**The recipe:** A queue holding nodes of the current level. Process exactly `len(queue)` nodes per level so you can group by depth.

```python
def levelOrder(root):
    if not root:
        return []
    queue = deque([root])
    result = []
    while queue:
        level = []
        for _ in range(len(queue)):     # only current level's nodes
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        result.append(level)
    return result
```

**Complexity:** O(n) time, O(w) space where w = max width (O(n) worst).

---

### 5.3 BST properties

**Trigger:** The tree is a **BST**. Then: in-order traversal yields sorted order; for any node, left subtree values < node < right subtree values. Use this to prune searches and to validate.

**Insight:** In a BST you rarely need to visit both children — the value range tells you which side to go. "Lowest common ancestor in a BST" is O(h) by comparing values, no need to track both subtrees.

---

## 6. GRAPHS

### 6.1 Graph DFS / BFS

**Trigger:** "number of islands," "clone graph," "word ladder," "connected components," "flood fill," "course schedule" (also see 6.3). Graphs are just trees with cycles, so you always need a **visited** set to avoid infinite loops.

**Core intuition:** DFS goes deep (recursion/stack) — good for connectivity, paths, cycle detection. BFS goes wide (queue) — good for shortest path in unweighted graphs. Both are O(V + E).

**Number of Islands (DFS + visited):**

```python
def numIslands(grid):
    if not grid:
        return 0
    m, n = len(grid), len(grid[0])
    visited = set()
    def dfs(r, c):
        if (r, c) in visited or not (0 <= r < m and 0 <= c < n) or grid[r][c] == '0':
            return
        visited.add((r, c))
        for dr, dc in [(1,0),(-1,0),(0,1),(0,-1)]:
            dfs(r + dr, c + dc)
    count = 0
    for r in range(m):
        for c in range(n):
            if grid[r][c] == '1' and (r, c) not in visited:
                dfs(r, c)
                count += 1          # finished one island
    return count
```

**BFS shortest path (unweighted):** Standard queue; first time you reach the target is the shortest distance.

**Complexity:** O(V + E) time, O(V) space.

**Common mistakes:**
- Forgetting the `visited` set → infinite recursion on cycles.
- Mutating the input grid in-place (writing `'2'`) instead of a separate visited set — both work, but be consistent and don't forget to mark.

---

### 6.2 Union Find (Disjoint Set Union)

**Trigger:** "connected components," "number of provinces," "redundant connection," "accounts merge," "dynamic connectivity" (edges added over time, ask "are these connected?"). Best when you have many *queries* about connectivity and edges arrive incrementally — faster and simpler than re-running DFS each time.

**Core intuition:** Each set has a representative (root). `find(x)` returns the root (with **path compression**: flatten the tree so future finds are fast). `union(x, y)` links the two roots, using **union by rank/size** so the tree stays shallow. Both operations are ~O(1) amortized (inverse-Ackermann, O(α(n))).

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n
        self.components = n
    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])   # path compression
        return self.parent[x]
    def union(self, x, y):
        px, py = self.find(x), self.find(y)
        if px == py:
            return False                  # already connected
        if self.rank[px] < self.rank[py]:
            self.parent[px] = py
        elif self.rank[px] > self.rank[py]:
            self.parent[py] = px
        else:
            self.parent[py] = px
            self.rank[px] += 1
        self.components -= 1
        return True
```

**Complexity:** O(α(n)) per operation, effectively O(1). Space O(n).

**Common mistakes:**
- Skipping path compression/union-by-rank → degenerate O(n) trees and TLE.
- Forgetting to decrement a component counter, then having to recount roots at the end (works but slower).

---

### 6.3 Topological Sort

**Trigger:** "course schedule," "build order," "task dependencies," "alien dictionary," "word break (ordering)." Any problem where items have **prerequisites** forming a DAG. The goal: order items so every prerequisite comes first, or detect that a cycle makes it impossible.

**Core intuition:** Two equivalent methods. **Kahn's (BFS):** repeatedly take nodes with indegree 0, emit them, and decrement neighbors' indegree. If you emit all nodes, it's a valid order; if not, there's a cycle. **DFS:** post-order traversal of a DAG yields a reverse topological order; a back-edge means a cycle.

```python
def canFinish(numCourses, prerequisites):
    graph = defaultdict(list)
    indegree = [0] * numCourses
    for dest, src in prerequisites:
        graph[src].append(dest)
        indegree[dest] += 1
    queue = deque(i for i in range(numCourses) if indegree[i] == 0)
    taken = 0
    while queue:
        node = queue.popleft()
        taken += 1
        for nb in graph[node]:
            indegree[nb] -= 1
            if indegree[nb] == 0:
                queue.append(nb)
    return taken == numCourses      # false ⇒ cycle ⇒ cannot finish
```

**Complexity:** O(V + E) time, O(V + E) space.

**Common mistakes:**
- Using DFS without the three-color visited state (0=unvisited, 1=in progress, 2=done) → can't distinguish a back-edge from a cross-edge, so cycle detection is wrong.

---

### 6.4 Shortest Path

| Situation | Algorithm | Complexity | Notes |
|-----------|-----------|------------|-------|
| Unweighted graph | BFS | O(V+E) | First visit = shortest |
| Weighted, non-negative | **Dijkstra** | O((V+E) log V) | Min-heap of (dist, node) |
| Weighted, negative edges OK | **Bellman-Ford** | O(VE) | Detects negative cycles |
| All-pairs | **Floyd-Warshall** | O(V³) | Small dense graphs |

**Dijkstra — why a heap:** Always expand the node with the currently smallest distance; the heap keeps that O(log V) per update. Skip stale entries (`if d > dist[node]: continue`).

```python
def dijkstra(graph, start):
    dist = {n: float('inf') for n in graph}
    dist[start] = 0
    pq = [(0, start)]
    while pq:
        d, node = heapq.heappop(pq)
        if d > dist[node]:
            continue
        for nb, w in graph[node]:
            nd = d + w
            if nd < dist[nb]:
                dist[nb] = nd
                heapq.heappush(pq, (nd, nb))
    return dist
```

**Common mistakes:**
- Using Dijkstra with negative weights → incorrect (use Bellman-Ford).
- Forgetting the stale-entry skip → duplicate work and wrong answers when an old distance is popped after a better one was already set.

---

## 7. DYNAMIC PROGRAMMING

DP is the pattern candidates fear most, usually because they memorize solutions instead of understanding the *shape* of the problem. Learn to recognize the shape and you can derive any DP.

### 7.1 When DP applies
A problem needs DP if it has **both**:
1. **Optimal substructure** — the best answer for `n` is built from the best answers for smaller parts.
2. **Overlapping subproblems** — you'd solve the same smaller part many times.

If those hold, DP beats plain recursion by storing results (memoization) or building bottom-up (tabulation).

**Two ways to build:**
- **Top-down (memoization):** recursive, cache results in a map. Easiest to write; follows the recursion you'd write anyway.
- **Bottom-up (tabulation):** fill a table from smallest to largest. Usually O(1) space-optimizable and avoids recursion-depth issues.

**The DP recipe:**
1. Define `dp[i]` (or `dp[i][j]`) in words: *"dp[i] = the answer for the prefix of length i."* A precise definition is 80% of the battle.
2. Find the recurrence: how does `dp[i]` relate to earlier entries?
3. Identify base cases (`dp[0]`, `dp[1]`).
4. Decide order (small → large) and what to return.
5. (Optional) Space-optimize if you only use a few previous rows.

---

### 7.2 1D DP

**Climbing Stairs** (Fibonacci shape — answer depends on previous 1–2 states):

```python
def climbStairs(n):
    if n <= 2:
        return n
    a, b = 1, 2
    for _ in range(3, n + 1):
        a, b = b, a + b      # rolling variables → O(1) space
    return b
```

**House Robber** (classic "take or skip"): `dp[i] = max(rob i + dp[i-2], skip i + dp[i-1])`.

```python
def rob(nums):
    if not nums:
        return 0
    if len(nums) <= 2:
        return max(nums)
    dp = [0] * len(nums)
    dp[0] = nums[0]
    dp[1] = max(nums[0], nums[1])
    for i in range(2, len(nums)):
        dp[i] = max(dp[i-1], dp[i-2] + nums[i])
    return dp[-1]
```

**Complexity:** O(n) time, O(n) → O(1) space after rolling optimization.

---

### 7.3 Grid / 2D DP

**Unique Paths** (`dp[i][j] = dp[i-1][j] + dp[i][j-1]`, base row/col = 1).
**Longest Increasing Path / Min Path Sum** follow the same "sum of top and left" shape.

```python
def uniquePaths(m, n):
    dp = [[1] * n for _ in range(m)]
    for i in range(1, m):
        for j in range(1, n):
            dp[i][j] = dp[i-1][j] + dp[i][j-1]
    return dp[m-1][n-1]
```

**Complexity:** O(mn) time, O(mn) → O(n) space (only previous row needed).

---

### 7.4 String DP

**Trigger:** "edit distance," "longest common subsequence," "longest palindromic subsequence," "regular expression / wildcard match." `dp[i][j]` = answer for `word1[:i]` and `word2[:j]`.

**Edit Distance** (insert/delete/replace):

```python
def minDistance(word1, word2):
    m, n = len(word1), len(word2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(m + 1):
        dp[i][0] = i          # delete all of word1
    for j in range(n + 1):
        dp[0][j] = j          # insert all of word2
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i-1] == word2[j-1]:
                dp[i][j] = dp[i-1][j-1]            # characters match, free
            else:
                dp[i][j] = 1 + min(dp[i-1][j],      # delete
                                   dp[i][j-1],      # insert
                                   dp[i-1][j-1])    # replace
    return dp[m][n]
```

**Complexity:** O(mn) time and space.

---

### 7.5 Interval & Knapsack DP

**0/1 Knapsack** ("can you pick items with weight ≤ W to maximize value?"): `dp[i][w] = max(dp[i-1][w], dp[i-1][w-weight[i]] + value[i])`. Space-optimize by iterating `w` **backwards**.

**Partition / palindrome partitioning:** `dp[i] =` can prefix `i` be partitioned into valid parts? Usually combined with a precomputed `isPalindrome[i][j]`.

**Common DP mistakes:**
- A vague `dp` definition → the recurrence never lands. Write the definition in a comment first.
- Wrong iteration direction in space-optimized knapsack (forward overwrites what you still need; go backward).
- Forgetting base cases → index errors or off-by-one in the table.

---

## 8. ADVANCED DATA STRUCTURES & TRICKS

### 8.1 Heap / Priority Queue

**Trigger:** "kth largest/smallest," "top K frequent," "median of a data stream," "merge k sorted lists," "tasks scheduled by priority." Heaps give O(1) peek at the extreme and O(log n) insert/remove.

- **Top K:** min-heap of size K; keep the K largest seen.
- **Median of stream:** two heaps — max-heap for the lower half, min-heap for the upper half; rebalance so sizes differ by ≤1.
- **Merge k lists:** min-heap of `(val, list_index, node)`; pop smallest, push its next. O(N log k).

```python
def findKthLargest(nums, k):
    return heapq.nlargest(k, nums)[-1]      # or: heapq.nsmallest for kth smallest
    # Manual: min-heap of size k, return heap[0]
```

**Complexity:** O(n log k) for top-K, O(log n) per heap op.

---

### 8.2 Quick Select & Divide & Conquer

**Quick Select** finds the kth element in **expected O(n)** (vs O(n log n) sorting). Like quicksort but you only recurse into the side containing the kth element.

```python
import random
def findKthLargest(nums, k):
    k_smallest = len(nums) - k          # 0-indexed position of kth largest
    def select(left, right):
        if left == right:
            return nums[left]
        pivot = random.randint(left, right)
        # partition so elements < pivot are on the left
        pivot = partition(left, right, pivot)
        if k_smallest == pivot:
            return nums[pivot]
        elif k_smallest < pivot:
            return select(left, pivot - 1)
        else:
            return select(pivot + 1, right)
    # (partition: move pivot to its sorted position, return its index)
    return select(0, len(nums) - 1)
```

**Complexity:** O(n) average, O(n²) worst (avoid with random pivot). Space O(1) (or O(log n) recursion).

---

### 8.3 Trie

**Trigger:** "autocomplete," "words starting with prefix," "search in a dictionary of words," "word break with many lookups." A Trie stores strings by shared prefixes; prefix queries become O(prefix length).

*(Reference implementation: see the Trie class — `insert`, `search`, `startsWith` — in the Anki/code appendix of the original notes; the structure is: each node has `children: dict` and `is_end: bool`.)*

**Complexity:** Insert/search O(L) where L = word length. Space O(total characters stored).

---

### 8.4 Bit Manipulation

**Trigger:** "single number appearing once (others twice/three times)," "power of two," "number of 1 bits," "subset generation via bitmask," "reverse bits."

**Core tricks:**
- `x & (x - 1)` → clears the lowest set bit. If result is 0, `x` is a power of two.
- `x ^ y` → bits differ. XOR all numbers → the one appearing once remains.
- `n & 1` → is `n` odd. `n >> 1` → divide by 2.
- Bitmask subsets: for `mask` in `range(1 << n)`, `mask & (1 << i)` tests if element `i` is in the subset (a compact way to iterate all subsets).

```python
def singleNumber(nums):
    res = 0
    for x in nums:
        res ^= x          # pairs cancel; the lone number survives
    return res
```

**Common mistakes:** Confusing `>>` (arithmetic/logical shift) with `/` for negatives in some languages; off-by-one in bitmask loops (`range(1 << n)` not `1 << (n-1)`).

---

## 9. ANTI-PATTERNS & CONFUSION TRAPS

These are the things that turn a known pattern into a failed interview. Read this section slowly.

| Trap | Why it confuses | The fix |
|------|-----------------|---------|
| **Pattern for pattern's sake** | Forcing sliding window on a subsequence problem | Re-read the keyword: *contiguous* ⇒ window; *subsequence* ⇒ two pointers/DP |
| **Wrong bound style in binary search** | Mixing `left<=right` with `right=mid` | Pick one style: inclusive (`<=`, `mid±1`) or exclusive (`<`, `right=mid`). Stay consistent. |
| **Off-by-one in window length** | `right-left` instead of `right-left+1` | Length of `[left,right]` inclusive is always `right-left+1` |
| **Not seeding prefix maps** | Missing subarrays starting at index 0 | Seed `freq = {0: 1}` before the loop |
| **Sharing the same list in backtracking** | All results end up identical | Append a *copy*: `result.append(path[:])` |
| **Forgetting `path.pop()`** | Only one branch explored | Always undo the choice after recursing |
| **No `visited` set in graph DFS** | Infinite loop on cycles | Mark visited before/after processing; check before recursing |
| **Dijkstra on negative weights** | Silent wrong answer | Use Bellman-Ford when edges can be negative |
| **Recursion depth on a linked-list tree** | Stack overflow | Use iterative/stack version for huge degenerate trees |
| **Kadane init `best=0`** | Fails on all-negative input | Init `best = cur = nums[0]` |
| **Reversing a list and losing `next`** | Broken links / lost nodes | Save `nxt = curr.next` before rewiring `curr.next` |
| **Mutating input unexpectedly** | Surprises the interviewer / breaks other calls | Prefer a separate `visited` set over overwriting the grid |

**The #1 confusion killer:** Before coding, *say your `dp`/`window`/`pointer` definition out loud.* If you can't state in one sentence what your state variable means, you're not ready to write the loop — and stating it prevents most off-by-one and base-case errors.

---

## 10. PRACTICE ROADMAP

### Decision tree (print this)
```
contiguous? ──yes──> Sliding Window
   │no
sorted? ──yes──> Two Pointers / Binary Search
   │no
subarray sum? ──yes──> Prefix Sum + Hash Map
   │no
next greater / histogram? ──yes──> Monotonic Stack
   │no
intervals / overlap? ──yes──> Sort + Merge
   │no
all combinations? ──yes──> Backtracking
   │no
linked list cycle/middle? ──yes──> Fast & Slow
   │no
linked list reverse/merge? ──yes──> Reversal / Dummy
   │no
tree level/shortest? ──yes──> BFS
   │no
tree path/ancestor? ──yes──> DFS
   │no
graph connectivity? ──yes──> DFS/BFS or Union Find
   │no
dependencies/order? ──yes──> Topological Sort
   │no
optimal + repeated subproblems? ──yes──> Dynamic Programming
   │no
kth / top-K / median stream? ──yes──> Heap or Quick Select
   │no
prefix matching? ──yes──> Trie
   │no
bitwise / XOR / power-of-two? ──yes──> Bit Manipulation
```

### Must-know problems (the "Blind 75" core)
| Pattern | Easy | Medium | Hard |
|---------|------|--------|------|
| Sliding Window | 643, 713 | 3, 76, 438 | 862 |
| Two Pointers | 125, 344 | 11, 15, 167 | 42 |
| Binary Search | 704, 744 | 33, 34, 153 | 4, 240 |
| Prefix Sum | 560, 724 | 523, 930 | 1074 |
| Monotonic Stack | 496, 739 | 84, 503 | 901 |
| Intervals | 252, 56 | 57, 435 | 218 |
| Backtracking | 78, 90 | 46, 77, 39 | 51 |
| Fast/Slow | 141, 876 | 142, 234 | 287 |
| Tree DFS | 104, 112 | 105, 236, 124 | 99 |
| Tree BFS | 102, 111 | 199 | 297 |
| Graph | 200, 133 | 207, 210, 743 | 269 |
| Union Find | 547, 684 | 685, 803 | 1135 |
| Topo Sort | 207, 210 | 269, 630 | 802 |
| DP 1D | 70, 198, 509 | 322, 139 | 123 |
| DP 2D | 62, 64 | 221, 174 | 85 |
| String DP | — | 114, 516 | 115, 97 |
| Heap | 215, 347 | 295, 355 | 358 |
| Quick Select | — | 215, 912 | — |
| Trie | 208, 211 | 212, 677 | 336 |
| Bits | 136, 191 | 201, 260 | 421 |

### Weekly study plan
- **Week 1:** Arrays & Strings (3.1–3.7). 2 problems/day, all Easy→Medium.
- **Week 2:** Linked Lists + Trees (4, 5). Focus on pointer/reversal and recursion.
- **Week 3:** Graphs + Union Find + Topo (6). Draw every graph by hand.
- **Week 4:** DP (7) — start with 1D, then grid, then string. Derive, don't memorize.
- **Week 5:** Advanced DS + mixed mock problems (8). Timed 25-min solves.

**Drill for every problem:** (1) name the pattern in 30s, (2) state the state definition, (3) code in 15 min, (4) give time/space in 30s.

---

### ANKI FLASHCARDS (reference)

| Front | Back |
|-------|------|
| Sliding Window trigger | Contiguous subarray/substring with a constraint |
| Two Pointers prerequisite | Array must be sorted (or sortable) |
| Binary search on answer | Use when answer is a number and "is X feasible?" is monotonic |
| Prefix sum seed | Start `freq = {0: 1}` so index-0 subarrays count |
| Monotonic stack use | Next greater/smaller, histogram, stock span |
| Interval first step | Sort by start time |
| Backtracking undo | `path.pop()` after recursing, append a copy to results |
| Fast/Slow cycle | O(1) space; reset one pointer to head to find cycle start |
| Reversal order | Save `nxt`, flip `curr.next`, advance both |
| Union Find speed | Path compression + union by rank → O(α(n)) |
| Topological sort | Kahn (BFS) or 3-color DFS; cycle ⇒ impossible |
| Dijkstra limit | Non-negative weights only |
| DP signal | Optimal substructure + overlapping subproblems |
| Knapsack loop dir | Iterate weight backwards when space-optimizing |
| Heap top-K | Min-heap of size K |
| Quick Select | Expected O(n), random pivot to avoid O(n²) |
| Trie use | Prefix matching / autocomplete |
| Bit trick | `x & (x-1)` clears lowest set bit; power-of-two if result 0 |
| Kadane init | `best = cur = nums[0]` (handles all-negative) |

---

> **Final interview wisdom:** The pattern is never the goal — *correct, communicated thinking* is. An interviewer would rather watch you derive a clean O(n) solution from first principles than recite a memorized template you can't explain. Know the patterns so recognition is instant, then slow down and show your reasoning. You've prepared. Trust it.
