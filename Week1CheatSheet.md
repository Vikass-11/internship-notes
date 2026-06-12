# Week 1 — Cheatsheet
**Vikas | Internship Week 1**

---

## Day 1 — Git & GitHub

**Fork vs Clone**
- Fork → copy someone's repo to my GitHub (when I don't have access)
- Clone → download repo to my laptop to work locally
- Team project → just clone, no fork needed

**Daily workflow:**
```bash
git checkout main
git pull origin main                     # always sync first
git checkout -b feature/name             # new branch for every task
# ... code ...
git add filename.java
git commit -m "feat: what I did"
git push -u origin feature/name          # first push
# open PR → review → merge
```

**Commands:**
```bash
git status                  # what changed
git log --oneline           # commit history
git diff                    # line changes
git fetch origin            # download, don't merge
git pull origin main        # download + merge
git restore filename        # discard changes
git reset --soft HEAD~1     # undo commit, keep files
git reset --hard HEAD~1     # undo commit + delete files ⚠️
git stash / git stash pop   # save & restore unfinished work
git revert <hash>           # safe undo — makes a new commit
git reflog                  # see every HEAD move — used to recover lost commits
```

**Commit types:** `feat` · `fix` · `refactor` · `docs` · `chore`

**Branch names:** `feature/name` · `fix/name` · `chore/name`

---

## Day 1 — SDLC

| # | Phase | What happens |
|---|---|---|
| 1 | Planning | Is it worth building? Timeline, cost, risks |
| 2 | Requirements | What must it do? (functional + non-functional) |
| 3 | Design | HLD (architecture) + LLD (DB, APIs, classes) |
| 4 | Implementation | Write code, open PRs — this is my phase |
| 5 | Testing | QA finds bugs, checks edge cases |
| 6 | Deployment | Local → Staging → Production |
| 7 | Maintenance | Fix bugs, add features, monitor |

**Models:**
- **Waterfall** → one phase at a time, no going back. For fixed projects.
- **Agile** → 2-week sprints, ship working software each sprint. Most companies use this.
- **Scrum** → Agile with roles (PO, Scrum Master, Dev) and ceremonies (standup, retro, review)
- **Kanban** → cards on a board: To Do → In Progress → Done

---

## Day 2 — HashMap

Key-value storage. Hash function converts key → bucket index → O(1) get/put/delete.

```java
HashMap<String, Integer> map = new HashMap<>();

map.put("alice", 95);
map.get("alice");                      // 95
map.getOrDefault("alice", 0);          // safe get, 0 if missing
map.containsKey("alice");              // true
map.remove("alice");

for (Map.Entry<String, Integer> e : map.entrySet())
    System.out.println(e.getKey() + " -> " + e.getValue());

// count frequency
for (char c : str.toCharArray())
    map.put(c, map.getOrDefault(c, 0) + 1);
```

**Patterns:**
```java
// Frequency count — anagram check
for (char c : s.toCharArray()) freq.put(c, freq.getOrDefault(c, 0) + 1);
for (char c : t.toCharArray()) freq.put(c, freq.getOrDefault(c, 0) - 1);
// all values should be 0 if anagram

// Complement lookup — two sum
// store seen values, check if (target - current) exists
int need = target - nums[i];
if (seen.containsKey(need)) return new int[]{seen.get(need), i};
seen.put(nums[i], i);
```

---

## Day 2 — Recursion

Function that calls itself on a smaller version of the same problem.
- **Base case** → when to stop
- **Recursive case** → call itself with smaller input

```java
// factorial
int factorial(int n) {
    if (n == 0) return 1;
    return n * factorial(n - 1);
}

// fibonacci — O(2^n) naive
int fib(int n) {
    if (n <= 1) return n;
    return fib(n-1) + fib(n-2);
}

// fibonacci with memo — O(n)
int fib(int n, HashMap<Integer,Integer> memo) {
    if (n <= 1) return n;
    if (memo.containsKey(n)) return memo.get(n);
    memo.put(n, fib(n-1, memo) + fib(n-2, memo));
    return memo.get(n);
}

// binary search
int bs(int[] arr, int t, int lo, int hi) {
    if (lo > hi) return -1;
    int mid = (lo + hi) / 2;
    if (arr[mid] == t) return mid;
    return arr[mid] < t ? bs(arr, t, mid+1, hi) : bs(arr, t, lo, mid-1);
}
```

**Memoization vs Tabulation:**
- Memoization → top-down, recursion + HashMap cache
- Tabulation → bottom-up, loop + array, no stack risk

---

## Day 3 — SQL

```sql
-- INNER JOIN  → only matching rows in both tables
-- LEFT JOIN   → all from left, NULL if no match on right
-- RIGHT JOIN  → all from right, NULL if no match on left
-- FULL JOIN   → everything from both

SELECT u.name, o.total
FROM users u LEFT JOIN orders o ON u.id = o.user_id;

-- group + filter
SELECT department, COUNT(*), AVG(salary)
FROM employees
GROUP BY department HAVING AVG(salary) > 60000;

-- transaction
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 1;
  UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT; -- or ROLLBACK if something fails
```

**Index** → speeds up lookups on a column. Without it → full table scan O(n). With it → O(log n).
```sql
CREATE INDEX idx_email ON users(email);
```

**ACID:**
- Atomicity → all steps succeed or none do
- Consistency → DB always stays valid
- Isolation → concurrent transactions don't interfere
- Durability → committed data survives crashes

---

## Day 3 — System Design

**Scalability:**
- Vertical → bigger machine. Has a limit.
- Horizontal → more machines. Preferred.

**Load Balancer** → splits traffic across servers. If one goes down, others handle it.

**Cache (Redis):**
```
request → check cache → hit: return fast | miss: query DB → store in cache → return
```
Hard part = knowing when to clear stale data (cache invalidation).

**CDN** → serves static files (images, JS, CSS) from servers near the user. Less load on backend.

**SQL vs NoSQL:**
- SQL → structured, joins, ACID, good for relational data
- NoSQL → flexible schema, scales horizontally, good for high volume

**Fault Tolerance:**
- Replication → copies of DB on multiple servers
- Failover → auto-switch to backup if primary dies

**CAP Theorem** → distributed system can only guarantee 2 of 3:
- Consistency, Availability, Partition Tolerance

---

## Quick Ref

```
Git:          pull → branch → code → commit → push → PR
SDLC:         Plan → Require → Design → Code → Test → Deploy → Maintain
HashMap:      O(1) | key → hash → bucket
Recursion:    base case + trust n-1
SQL:          INNER/LEFT/RIGHT/FULL joins | ACID transactions | index for speed
System:       load balance → cache → DB | replicate data | scale horizontal
```
