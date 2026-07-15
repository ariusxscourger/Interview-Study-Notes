# CODING PATTERNS FOR INTERVIEWS
## Exam-Style Study Notes

---

## TOPIC: Algorithmic Patterns for Coding Interviews

### WHY? (Problem Statement)

**Coding interviews test:**
- Problem-solving ability
- Code quality and structure
- Time/space complexity analysis
- Communication during problem-solving

**What Pattern Recognition Solves:**
| Problem | Solution |
|---------|----------|
| Blank screen syndrome | Recognize pattern → apply template |
| Time pressure | Known time/space complexities |
| Edge cases | Pattern includes standard edge cases |
| Optimization | Know when to optimize |

---

### HOW? (Pattern Recognition Framework)

```
PROBLEM → IDENTIFY PATTERN → APPLY TEMPLATE → OPTIMIZE → TEST
```

#### **Pattern Recognition Checklist:**

```
ARRAY/STRING:
□ Fixed window? → Sliding Window
□ Subarray sum/target? → Prefix Sum / Hash Map
□ Sorted? → Two Pointers / Binary Search
□ Subsequence? → Two Pointers / DP
□ Partition? → Quick Select / Dutch Flag

LINKED LIST:
□ Cycle? → Floyd's Tortoise & Hare
□ Reverse? → Iterative 3-pointer
□ Merge? → Merge Sort logic
□ Palindrome? → Fast/Slow + Reverse half

TREE/GRAPH:
□ Path? → DFS
□ Shortest path? → BFS
□ Connected components? → Union Find / DFS
□ Topological order? → Kahn's / DFS
□ Cycle? → DFS + visited states
□ MST? → Kruskal / Prim

DYNAMIC PROGRAMMING:
□ Optimal substructure? → DP
□ Overlapping subproblems? → Memoization / Tabulation
□ Sequence? → 1D DP
□ Grid? → 2D DP
□ Partition? → Knapsack / Partition DP
```

---

### WHAT? (Core Patterns with Templates)

#### 1. **Sliding Window** (Array/String, O(n))

```python
# Fixed Window Size
def maxSumSubarray(nums, k):
    window_sum = sum(nums[:k])
    max_sum = window_sum
    for i in range(k, len(nums)):
        window_sum += nums[i] - nums[i - k]
        max_sum = max(max_sum, window_sum)
    return max_sum

# Variable Window (Condition-based)
def minSubArrayLen(target, nums):
    left = 0
    current_sum = 0
    min_len = float('inf')
    for right in range(len(nums)):
        current_sum += nums[right]
        while current_sum >= target:
            min_len = min(min_len, right - left + 1)
            current_sum -= nums[left]
            left += 1
    return min_len if min_len != float('inf') else 0
```

#### 2. **Two Pointers** (Sorted Array, O(n))

```python
# Two Sum (Sorted)
def twoSumSorted(nums, target):
    left, right = 0, len(nums) - 1
    while left < right:
        current = nums[left] + nums[right]
        if current == target: return [left, right]
        elif current < target: left += 1
        else: right -= 1
    return []

# Remove Duplicates (In-place)
def removeDuplicates(nums):
    if not nums: return 0
    write = 1
    for read in range(1, len(nums)):
        if nums[read] != nums[write - 1]:
            nums[write] = nums[read]
            write += 1
    return write

# Container With Most Water
def maxArea(height):
    left, right = 0, len(height) - 1
    max_area = 0
    while left < right:
        area = min(height[left], height[right]) * (right - left)
        max_area = max(max_area, area)
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1
    return max_area
```

#### 3. **Fast & Slow Pointers** (Linked List, O(n))

```python
# Cycle Detection
def hasCycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast: return True
    return False

# Find Cycle Start
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

# Middle of Linked List
def middleNode(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    return slow
```

#### 4. **Binary Search** (O(log n))

```python
# Standard Binary Search
def binarySearch(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = left + (right - left) // 2
        if nums[mid] == target: return mid
        elif nums[mid] < target: left = mid + 1
        else: right = mid - 1
    return -1

# Lower Bound (First >= target)
def lowerBound(nums, target):
    left, right = 0, len(nums)
    while left < right:
        mid = left + (right - left) // 2
        if nums[mid] < target: left = mid + 1
        else: right = mid
    return left

# Search in Rotated Sorted Array
def searchRotated(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = (left + right) // 2
        if nums[mid] == target: return mid
        if nums[left] <= nums[mid]:
            if nums[left] <= target < nums[mid]: right = mid - 1
            else: left = mid + 1
        else:
            if nums[mid] < target <= nums[right]: left = mid + 1
            else: right = mid - 1
    return -1
```

#### 5. **Prefix Sum / Hash Map** (Subarray Problems, O(n))

```python
# Subarray Sum Equals K
def subarraySum(nums, k):
    count = 0
    prefix_sum = 0
    freq = {0: 1}
    for num in nums:
        prefix_sum += num
        if prefix_sum - k in freq:
            count += freq[prefix_sum - k]
        freq[prefix_sum] = freq.get(prefix_sum, 0) + 1
    return count

# Maximum Subarray Sum (Kadane's Algorithm)
def maxSubArray(nums):
    max_sum = current_sum = nums[0]
    for num in nums[1:]:
        current_sum = max(num, current_sum + num)
        max_sum = max(max_sum, current_sum)
    return max_sum

# Longest Substring Without Repeating Characters
def lengthOfLongestSubstring(s):
    seen = {}
    left = 0
    max_len = 0
    for right, char in enumerate(s):
        if char in seen and seen[char] >= left:
            left = seen[char] + 1
        seen[char] = right
        max_len = max(max_len, right - left + 1)
    return max_len
```

#### 6. **Monotonic Stack** (Next Greater/Smaller, O(n))

```python
# Next Greater Element
def nextGreaterElement(nums):
    stack = []
    result = [-1] * len(nums)
    for i, num in enumerate(nums):
        while stack and nums[stack[-1]] < num:
            idx = stack.pop()
            result[idx] = num
        stack.append(i)
    return result

# Daily Temperatures
def dailyTemperatures(temperatures):
    stack = []
    result = [0] * len(temperatures)
    for i, temp in enumerate(temperatures):
        while stack and temperatures[stack[-1]] < temp:
            idx = stack.pop()
            result[idx] = i - idx
        stack.append(i)
    return result

# Largest Rectangle in Histogram
def largestRectangleArea(heights):
    stack = []
    max_area = 0
    for i, h in enumerate(heights + [0]):
        while stack and heights[stack[-1]] > h:
            height = heights[stack.pop()]
            width = i if not stack else i - stack[-1] - 1
            max_area = max(max_area, height * width)
        stack.append(i)
    return max_area
```

#### 7. **Union Find (Disjoint Set)** (Connected Components, O(α(n)))

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n
        self.count = n
    
    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]
    
    def union(self, x, y):
        px, py = self.find(x), self.find(y)
        if px == py: return False
        if self.rank[px] < self.rank[py]:
            self.parent[px] = py
        elif self.rank[px] > self.rank[py]:
            self.parent[py] = px
        else:
            self.parent[py] = px
            self.rank[px] += 1
        self.count -= 1
        return True

# Number of Islands
def numIslands(grid):
    if not grid: return 0
    m, n = len(grid), len(grid[0])
    uf = UnionFind(m * n)
    
    for i in range(m):
        for j in range(n):
            if grid[i][j] == '1':
                for di, dj in [(0,1),(1,0)]:
                    ni, nj = i + di, j + dj
                    if 0 <= ni < m and 0 <= nj < n and grid[ni][nj] == '1':
                        uf.union(i * n + j, ni * n + nj)
    
    return sum(1 for i in range(m) for j in range(n) 
               if grid[i][j] == '1' and uf.find(i * n + j) == i * n + j)
```

#### 8. **Topological Sort** (DAG, Course Schedule)

```python
# Kahn's Algorithm (BFS)
def topologicalSort(numCourses, prerequisites):
    graph = defaultdict(list)
    indegree = [0] * numCourses
    
    for dest, src in prerequisites:
        graph[src].append(dest)
        indegree[dest] += 1
    
    queue = deque([i for i in range(numCourses) if indegree[i] == 0])
    order = []
    
    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph[node]:
            indegree[neighbor] -= 1
            if indegree[neighbor] == 0:
                queue.append(neighbor)
    
    return order if len(order) == numCourses else []

# DFS Approach (Cycle Detection)
def canFinish(numCourses, prerequisites):
    graph = defaultdict(list)
    for dest, src in prerequisites:
        graph[src].append(dest)
    
    visited = [0] * numCourses  # 0=unvisited, 1=visiting, 2=visited
    
    def hasCycle(node):
        if visited[node] == 1: return True
        if visited[node] == 2: return False
        visited[node] = 1
        for neighbor in graph[node]:
            if hasCycle(neighbor): return True
        visited[node] = 2
        return False
    
    for i in range(numCourses):
        if hasCycle(i): return False
    return True
```

#### 9. **Dynamic Programming** (Bottom-Up)

```python
# Climbing Stairs (Fibonacci)
def climbStairs(n):
    if n <= 2: return n
    a, b = 1, 2
    for _ in range(3, n + 1):
        a, b = b, a + b
    return b

# Coin Change (Min Coins)
def coinChange(coins, amount):
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0
    for i in range(1, amount + 1):
        for coin in coins:
            if coin <= i:
                dp[i] = min(dp[i], dp[i - coin] + 1)
    return dp[amount] if dp[amount] != float('inf') else -1

# Longest Increasing Subsequence (O(n log n))
def lengthOfLIS(nums):
    tails = []
    for num in nums:
        idx = bisect_left(tails, num)
        if idx == len(tails):
            tails.append(num)
        else:
            tails[idx] = num
    return len(tails)

# House Robber
def rob(nums):
    if not nums: return 0
    if len(nums) <= 2: return max(nums)
    dp = [0] * len(nums)
    dp[0], dp[1] = nums[0], max(nums[0], nums[1])
    for i in range(2, len(nums)):
        dp[i] = max(dp[i-1], dp[i-2] + nums[i])
    return dp[-1]

# Edit Distance
def minDistance(word1, word2):
    m, n = len(word1), len(word2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(m + 1): dp[i][0] = i
    for j in range(n + 1): dp[0][j] = j
    
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i-1] == word2[j-1]:
                dp[i][j] = dp[i-1][j-1]
            else:
                dp[i][j] = 1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
    return dp[m][n]
```

#### 10. **Tree Traversal Patterns**

```python
# BFS (Level Order)
def levelOrder(root):
    if not root: return []
    queue = deque([root])
    result = []
    while queue:
        level = []
        for _ in range(len(queue)):
            node = queue.popleft()
            level.append(node.val)
            if node.left: queue.append(node.left)
            if node.right: queue.append(node.right)
        result.append(level)
    return result

# DFS (Pre/In/Post Order)
def inorderTraversal(root):
    result = []
    def inorder(node):
        if not node: return
        inorder(node.left)
        result.append(node.val)
        inorder(node.right)
    inorder(root)
    return result

# Iterative (using stack)
def inorderIterative(root):
    result, stack = [], []
    curr = root
    while curr or stack:
        while curr:
            stack.append(curr)
            curr = curr.left
        curr = stack.pop()
        result.append(curr.val)
        curr = curr.right
    return result

# Path Sum
def hasPathSum(root, targetSum):
    if not root: return False
    if not root.left and not root.right:
        return root.val == targetSum
    return (hasPathSum(root.left, targetSum - root.val) or 
            hasPathSum(root.right, targetSum - root.val))

# Lowest Common Ancestor
def lowestCommonAncestor(root, p, q):
    if not root or root == p or root == q:
        return root
    left = lowestCommonAncestor(root.left, p, q)
    right = lowestCommonAncestor(root.right, p, q)
    if left and right: return root
    return left or right
```

#### 11. **Graph Algorithms**

```python
# Dijkstra (Shortest Path)
def dijkstra(graph, start):
    dist = {node: float('inf') for node in graph}
    dist[start] = 0
    pq = [(0, start)]
    
    while pq:
        d, node = heapq.heappop(pq)
        if d > dist[node]: continue
        for neighbor, weight in graph[node]:
            new_dist = d + weight
            if new_dist < dist[neighbor]:
                dist[neighbor] = new_dist
                heapq.heappush(pq, (new_dist, neighbor))
    return dist

# Bellman-Ford (Negative Weights)
def bellmanFord(edges, n, src):
    dist = [float('inf')] * n
    dist[src] = 0
    for _ in range(n - 1):
        for u, v, w in edges:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
    # Check negative cycle
    for u, v, w in edges:
        if dist[u] + w < dist[v]: return None
    return dist

# Floyd-Warshall (All Pairs)
def floydWarshall(graph):
    n = len(graph)
    dist = [[graph[i][j] for j in range(n)] for i in range(n)]
    for k in range(n):
        for i in range(n):
            for j in range(n):
                dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j])
    return dist
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | When to Use | Anti-Pattern |
|---------|-------------|--------------|
| Sliding Window | Subarray/substring with constraint | Nested loops O(n²) |
| Two Pointers | Sorted array, pair problems | Nested loops |
| Fast/Slow | Cycle detection, middle, palindrome | Hash set for cycle (O(n) space) |
| Binary Search | Sorted array, monotonic function | Linear search |
| Prefix Sum/Hash Map | Subarray sum, frequency | Nested loops |
| Monotonic Stack | Next greater/smaller, histogram | Nested loops |
| Union Find | Connected components, dynamic connectivity | DFS every query |
| Topological Sort | Dependency ordering, course schedule | DFS without cycle detection |
| DP | Optimal substructure + overlapping | Recursion without memo |
| BFS/DFS | Graph traversal, shortest path (unweighted) | Wrong choice for problem |

---

### INTERVIEW QUESTIONS (Must-Know Problems)

| Problem | Pattern | Key Insight |
|---------|---------|-------------|
| Two Sum | Hash Map | O(n) with complement lookup |
| 3Sum | Two Pointers | Sort + two pointers, skip duplicates |
| Container With Most Water | Two Pointers | Greedy: move smaller height |
| Longest Substring w/o Repeat | Sliding Window + Hash Map | Track char indices |
| Minimum Window Substring | Sliding Window + Counter | Expand right, shrink left |
| Valid Parentheses | Stack | Push open, pop on close |
| Merge Intervals | Sort + Merge | Sort by start, merge overlap |
| Merge k Sorted Lists | Heap / Divide & Conquer | Min-heap of heads |
| LRU Cache | Hash Map + Doubly Linked List | O(1) get/put |
| Median of Data Stream | Two Heaps | Max heap (lower), min heap (upper) |
| Kth Largest Element | Quick Select / Min Heap | O(n) avg / O(n log k) |
| Course Schedule | Topological Sort | BFS (Kahn) or DFS |
| Number of Islands | Union Find / DFS / BFS | Mark visited, count components |
| Clone Graph | BFS/DFS + Hash Map | Old node → new node mapping |
| Word Ladder | BFS + Pattern Matching | Bi-directional BFS for speed |
| Serialize/Deserialize Tree | BFS/DFS + String | Pre-order with null markers |

---

### GOTCHAS & EDGE CASES

- [ ] **Empty input** - Always check `if not nums: return 0`
- [ ] **Single element** - `len(nums) == 1`
- [ ] **Duplicates** - Skip in 3Sum, handle in BST
- [ ] **Overflow** - Python handles, but mention in other languages
- [ ] **Negative numbers** - Kadane's, binary search
- [ ] **Large input** - O(n²) → TLE, need O(n log n) or O(n)
- [ ] **Recursion depth** - Python limit 1000, use iterative
- [ ] **Mutable defaults** - `def f(arr=[])` → bug, use `None`
- [ ] **Index bounds** - `left <= right` vs `left < right`
- [ ] **Integer division** - `//` vs `/` in Python 3

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Quick Select (Kth Largest - O(n) avg)**

```python
def findKthLargest(nums, k):
    def quickSelect(left, right, k_smallest):
        if left == right: return nums[left]
        pivot_idx = random.randint(left, right)
        pivot_idx = partition(left, right, pivot_idx)
        if k_smallest == pivot_idx:
            return nums[k_smallest]
        elif k_smallest < pivot_idx:
            return quickSelect(left, pivot_idx - 1, k_smallest)
        else:
            return quickSelect(pivot_idx + 1, right, k_smallest)
    
    def partition(left, right, pivot_idx):
        pivot = nums[pivot_idx]
        nums[pivot_idx], nums[right] = nums[right], nums[pivot_idx]
        store_idx = left
        for i in range(left, right):
            if nums[i] < pivot:
                nums[store_idx], nums[i] = nums[i], nums[store_idx]
                store_idx += 1
        nums[right], nums[store_idx] = nums[store_idx], nums[right]
        return store_idx
    
    return quickSelect(0, len(nums) - 1, len(nums) - k)
```

#### 2. **Trie Implementation**

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False

class Trie:
    def __init__(self):
        self.root = TrieNode()
    
    def insert(self, word):
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
        node.is_end = True
    
    def search(self, word):
        node = self.root
        for char in word:
            if char not in node.children: return False
            node = node.children[char]
        return node.is_end
    
    def startsWith(self, prefix):
        node = self.root
        for char in prefix:
            if char not in node.children: return False
            node = node.children[char]
        return True
```

#### 3. **Segment Tree (Range Sum Query)**

```python
class SegmentTree:
    def __init__(self, nums):
        self.n = len(nums)
        self.tree = [0] * (4 * self.n)
        self.build(nums, 0, 0, self.n - 1)
    
    def build(self, nums, node, start, end):
        if start == end:
            self.tree[node] = nums[start]
        else:
            mid = (start + end) // 2
            self.build(nums, 2*node+1, start, mid)
            self.build(nums, 2*node+2, mid+1, end)
            self.tree[node] = self.tree[2*node+1] + self.tree[2*node+2]
    
    def query(self, node, start, end, l, r):
        if r < start or end < l: return 0
        if l <= start and end <= r: return self.tree[node]
        mid = (start + end) // 2
        return self.query(2*node+1, start, mid, l, r) + \
               self.query(2*node+2, mid+1, end, l, r)
    
    def update(self, node, start, end, idx, val):
        if start == end:
            self.tree[node] = val
        else:
            mid = (start + end) // 2
            if idx <= mid:
                self.update(2*node+1, start, mid, idx, val)
            else:
                self.update(2*node+2, mid+1, end, idx, val)
            self.tree[node] = self.tree[2*node+1] + self.tree[2*node+2]
```

---

### PRACTICE PROBLEMS (LeetCode Numbers)

| Pattern | Easy | Medium | Hard |
|---------|------|--------|------|
| Sliding Window | 643, 713 | 3, 76, 438 | 727, 862 |
| Two Pointers | 125, 344 | 11, 15, 167 | 42, 632 |
| Fast/Slow | 141, 876 | 142, 234 | 287 |
| Binary Search | 704, 744 | 33, 34, 153 | 4, 240 |
| Prefix Sum | 560, 724 | 523, 525, 930 | 1590, 1746 |
| Monotonic Stack | 496, 739 | 84, 496, 503 | 84, 901, 907 |
| Union Find | 547, 684 | 685, 803, 1135 | 1135 |
| Topo Sort | 207, 210 | 269, 630 | 802 |
| DP | 70, 198, 509 | 198, 322, 300 | 123, 188, 312 |
| Tree BFS/DFS | 102, 104, 111 | 101, 112, 199 | 124, 236, 297 |
| Graph | 200, 207 | 207, 210, 743 | 269, 332, 787 |
| Heap | 215, 347 | 295, 355 | 358, 480 |

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Sliding Window Use Case | Subarray/substring with constraint (sum, unique chars, max/min) |
| Two Pointers Prerequisite | Array MUST be sorted |
| Fast/Slow Pointer Cycle Detection | O(1) space, meets at cycle start |
| Binary Search Template | `while left <= right: mid = (l+r)//2` |
| Prefix Sum Use Case | Subarray sum, range sum queries |
| Monotonic Stack Use Case | Next greater/smaller, histogram, stock span |
| Union Find Complexity | O(α(n)) ≈ O(1) amortized |
| Topological Sort | Kahn's (BFS) or DFS, detect cycles |
| DP Optimal Substructure | Problem can be broken into optimal subproblems |
| DP Overlapping Subproblems | Same subproblems solved repeatedly |
| Trie Use Case | Prefix matching, autocomplete, dictionary |
| Segment Tree | Range query + point update, O(log n) |
| Quick Select | Kth element, O(n) average, O(n²) worst |
| Cycle Detection | Floyd's (fast/slow) O(1) space |
| Dijkstra | Non-negative weights, O((V+E) log V) |
| Bellman-Ford | Negative weights OK, O(VE) |
| Floyd-Warshall | All-pairs shortest path, O(V³) |

---

## NEXT TOPIC: `04-NEGOTIATION.md`

> **Study Tip**: Do 1 problem per pattern per day. Focus on: 1) Recognize pattern in 30 seconds, 2) Write clean code in 15 min, 3) Explain time/space in 30 sec.