# CS G516 — Advanced Database Systems
## 50 More Worked Examples: Query Plan Generation, Optimization & Cost Calculation
**Institution:** BITS-Pilani | **Instructor:** Dr. Yashvardhan Sharma | **Classes:** 6, 7, 8

---

## 📐 Schema & Parameters (Reference)

```
Relations:
  Student  (sid*, sname, age, gpa)         NR=1000,  LR=50B,  FR=10,  BR=100
  Takes    (sid,  cid,   grade)            NR=5000,  LR=20B,  FR=25,  BR=200
  Course   (cid*, cname, dept, credits)    NR=100,   LR=40B,  FR=12,  BR=9
  Employee (eid*, ename, dno, salary, age) NR=2000,  LR=50B,  FR=10,  BR=200
  Department(dno*, dname, mgreid, budget)  NR=50,    LR=30B,  FR=17,  BR=3
  Project  (pno*, pname, dno, budget)      NR=200,   LR=40B,  FR=12,  BR=17
  Supplier (sup_id*, sname, city, rating)  NR=500,   LR=40B,  FR=12,  BR=42
  Parts    (pid*, pname, color, weight)    NR=800,   LR=30B,  FR=17,  BR=48
  Supply   (sup_id, pid, qty, price)       NR=10000, LR=20B,  FR=25,  BR=400

Indexes (B+ Trees):
  Primary  on Student.sid      HT=2    Primary  on Course.cid       HT=1
  Primary  on Employee.eid     HT=2    Primary  on Department.dno   HT=1
  Primary  on Supplier.sup_id  HT=2    Primary  on Parts.pid        HT=2
  Secondary on Student.gpa     HT=2    Secondary on Student.age     HT=2
  Secondary on Employee.dno    HT=2    Secondary on Employee.salary  HT=2
  Secondary on Employee.age    HT=2    Secondary on Supply.pid      HT=3
  Secondary on Supply.sup_id   HT=3    Secondary on Parts.color     HT=2

Distinct Values V(A,R):
  V(gpa, Student)      = 20      V(age, Student)       = 50
  V(dno, Employee)     = 10      V(salary, Employee)   = 500
  V(age, Employee)     = 40      V(dept, Course)       = 5
  V(credits, Course)   = 4       V(cid, Takes)         = 100
  V(sid, Takes)        = 1000    V(grade, Takes)       = 5
  V(city, Supplier)    = 20      V(rating, Supplier)   = 10
  V(color, Parts)      = 8       V(weight, Parts)      = 50
  V(qty, Supply)       = 100     V(price, Supply)      = 200
  V(dno, Project)      = 10      V(budget, Project)    = 50

Selection Cardinalities SC(A,R) = ceil(NR / V(A,R)):
  SC(sid, Student)     = 1  (key)  SC(gpa, Student)    = 50
  SC(age, Student)     = 20        SC(eid, Employee)   = 1  (key)
  SC(dno, Employee)    = 200       SC(salary,Employee) = 4
  SC(age, Employee)    = 50        SC(cid, Course)     = 1  (key)
  SC(dept, Course)     = 20        SC(credits, Course) = 25
  SC(grade, Takes)     = 1000      SC(cid, Takes)      = 50
  SC(sid, Takes)       = 5         SC(sup_id,Supplier) = 1  (key)
  SC(city, Supplier)   = 25        SC(rating,Supplier) = 50
  SC(pid, Parts)       = 1  (key)  SC(color, Parts)    = 100
  SC(weight,Parts)     = 16        SC(pid, Supply)     = 400
  SC(sup_id, Supply)   = 20        SC(pno, Project)    = 1  (key)

Cost Parameters:
  tT = 0.1 ms  (time to transfer one block)
  tS = 4.0 ms  (time for one disk seek)
  Cost = b × tT + S × tS
```

---

> **How to read each example:**
> 1. **SQL** — the original query
> 2. **Initial RA** — canonical (unoptimized) relational algebra
> 3. **Optimized RA** — after applying heuristic rules
> 4. **Optimization Steps** — which rules were applied and why
> 5. **Cost Calculation** — numeric cost for each applicable algorithm
> 6. **Cost Table** — side-by-side comparison with ★ = best choice
> 7. **Key Insight** — the critical lesson from this example

---

## SECTION A — Selection Algorithm Deep Dives (Examples 31–40)

---

### Example 31: Binary Search vs Index on Ordered File

**SQL Query:**
```sql
SELECT * FROM Employee WHERE salary = 75000;
```

**Initial RA:**
```
σ_{salary=75000}(Employee)
```

**Optimized RA:**
```
A2/A4 via secondary B+tree index on salary (HT=2)
  -- if few matches: index faster than full scan
```

**Optimization Steps:**

- **Step 1:** salary is NOT a key. SC(salary, Employee) = ceil(2000/500) = **4 matching tuples**.
- **Step 2:** Secondary B+tree on salary (HT=2) exists. With only 4 matches, index is ideal.
- **Step 3:** Compare A1 (linear), A4 (secondary index), and hypothetical binary search.
- **Step 4:** Binary search NOT generally applicable — data not physically sorted; requires more seeks than an index.
- **Step 5:** A4 wins because n=4 is tiny (very high selectivity).

**Cost Calculation:**

```
A1 — Linear Scan:
  Cost = BR × tT + tS
       = 200 × 0.1 + 4
       = 20 + 4 = 24.0 ms

Binary Search (hypothetical, if file were sorted on salary):
  Cost = ceil(log2(BR)) × (tT + tS)
       = ceil(log2(200)) × (0.1 + 4)
       = 8 × 4.1 = 32.8 ms
  NOTE: Still need to scan b = ceil(SC/FR) = ceil(4/10) = 1 block of results
  Total = 32.8 + 1×4.1 = 36.9 ms   ← WORSE than A1!

A4 — Secondary B+tree, Equality on Non-Key (n=4 records):
  Cost = (HT + n) × (tT + tS)
       = (2 + 4) × (0.1 + 4)
       = 6 × 4.1 = 24.6 ms
```

**Cost Comparison Table:**

| Algorithm | Formula | Result (ms) | Notes |
|-----------|---------|-------------|-------|
| A1 Linear Scan | `BR×tT + tS` | **24.0 ★** | Wins by small margin |
| Binary Search | `ceil(log2(BR))×(tT+tS)` | 36.9 | More seeks than index |
| A4 Secondary Index | `(HT+n)×(tT+tS)` | 24.6 | Nearly tied with scan |

> **Key Insight:** Binary search is RARELY better than a B+tree index — the index provides O(log n) in tree height (typically 2–3 levels) while binary search on blocks also requires O(log n) seeks but without the benefit of a prebuilt structure. A1 wins here because n=4 means the file is sparsely matching and the single seek amortizes over all 200 blocks.

---

### Example 32: Primary Index on Non-Key vs Linear Scan — Break-Even Point

**SQL Query:**
```sql
SELECT * FROM Employee WHERE dno = 7;
-- Assume Employee IS clustered/sorted on dno (clustering index, HT=2)
```

**Initial RA:**
```
σ_{dno=7}(Employee)
```

**Optimized RA:**
```
A3: Clustering B+tree on dno — retrieve consecutive blocks
```

**Optimization Steps:**

- **Step 1:** SC(dno=7, Employee) = 200 matching tuples.
- **Step 2:** b = ceil(SC/FR) = ceil(200/10) = **20 consecutive blocks** of results (clustering index).
- **Step 3:** Calculate break-even: when does A3 stop being worth it vs A1?
- **Step 4:** Break-even when A3 cost = A1 cost → find the threshold SC value.

**Cost Calculation:**

```
A1 — Linear Scan:
  Cost = BR × tT + tS = 200×0.1 + 4 = 24.0 ms

A3 — Clustering Index, Equality on Non-Key (HT=2):
  b = ceil(SC/FR) = ceil(200/10) = 20 data blocks
  Cost = HT×(tT+tS) + tS + tT×b
       = 2×(0.1+4) + 4 + 0.1×20
       = 8.2 + 4 + 2.0 = 14.2 ms   ← BETTER

Break-Even Analysis:
  Set A3 cost = A1 cost:
  HT×(tT+tS) + tS + tT×b = BR×tT + tS
  2×4.1 + 4 + 0.1×b = 200×0.1 + 4
  8.2 + 4 + 0.1b = 20 + 4
  0.1b = 11.8
  b = 118 blocks → SC = 118×FR = 118×10 = 1180 tuples

  So: if SC > 1180 (> 59% of the file), use A1. Otherwise A3 wins.
  Here SC=200 → 10% of file → A3 clearly wins.
```

**Cost Comparison Table:**

| Algorithm | Cost (ms) | When to use |
|-----------|-----------|-------------|
| A3 Clustering Index ★ | **14.2 ★** | SC ≤ 1180 (≤59% of file) |
| A1 Linear Scan | 24.0 | SC > 1180 (>59% of file) |

> **Key Insight:** The break-even point for clustering index vs linear scan is approximately **59% of the file** for this example. For secondary (non-clustering) indexes, the break-even is MUCH lower (often 5–10%) due to random I/Os.

---

### Example 33: Multiple Conditions — Choosing the Right Single Index (A7 Deep Dive)

**SQL Query:**
```sql
SELECT * FROM Employee WHERE dno=2 AND age=30 AND salary=65000;
```

**Initial RA:**
```
σ_{dno=2 ∧ age=30 ∧ salary=65000}(Employee)
```

**Optimized RA:**
```
A7: Use most selective index (salary, SC=4), apply other conditions in memory
```

**Optimization Steps:**

- **Step 1:** Calculate SC for each condition independently:
  - SC(dno=2) = 200 tuples (least selective)
  - SC(age=30) = 50 tuples
  - SC(salary=65000) = 4 tuples (**most selective**)
- **Step 2:** Secondary indexes exist on dno(HT=2), salary(HT=2), age(HT=2).
- **Step 3:** A7 rule: pick the index with **lowest cost AND fewest tuples**.
- **Step 4:** Use salary index (n=4), fetch 4 records, test dno and age in memory.
- **Step 5:** Compare with A1 to verify index is worth it.

**Cost Calculation:**

```
A1 — Linear Scan:
  Cost = 200×0.1 + 4 = 24.0 ms

A7 using dno index (SC=200, worst):
  Cost = (HT + n) × (tT+tS) = (2+200)×4.1 = 828.2 ms   ← terrible

A7 using age index (SC=50):
  Cost = (2+50)×4.1 = 213.2 ms   ← still bad

A7 using salary index (SC=4) — BEST:
  Cost = (HT + n) × (tT+tS) = (2+4)×4.1 = 24.6 ms
  Apply dno=2 and age=30 in memory on 4 fetched records → free
  Total = 24.6 ms

Selectivity Comparison:
  RF(salary=65000) = 4/2000 = 0.002   ← most selective
  RF(age=30)       = 50/2000 = 0.025
  RF(dno=2)        = 200/2000 = 0.1   ← least selective
```

**Cost Comparison Table:**

| Index Used | SC | Cost Formula | Cost (ms) | |
|------------|-----|-------------|-----------|--|
| A1 Linear Scan | 2000 | `BR×tT+tS` | 24.0 | |
| A7 via dno index | 200 | `(2+200)×4.1` | 828.2 | ❌ |
| A7 via age index | 50 | `(2+50)×4.1` | 213.2 | |
| A7 via salary index ★ | 4 | `(2+4)×4.1` | **24.6 ★** | Best index choice |

> **Key Insight:** In A7, **always use the index on the most selective condition** (smallest SC/RF). Using the wrong index (dno with SC=200) is 34× slower than using the right one (salary with SC=4). However, A1 (24.0ms) ties with the best index choice here — secondary index overhead can negate the benefit.

---

### Example 34: Range Query with Secondary Index — When NOT to Use Index

**SQL Query:**
```sql
SELECT * FROM Employee WHERE salary BETWEEN 40000 AND 120000;
```

**Initial RA:**
```
σ_{salary≥40000 ∧ salary≤120000}(Employee)
```

**Optimized RA:**
```
A1: Linear scan — range covers too many tuples for secondary index
```

**Optimization Steps:**

- **Step 1:** Estimate qualifying tuples. Range = 40k–120k out of (say) 20k–200k total range → RF ≈ (120-40)/(200-20) = 80/180 ≈ **44% of employees**.
- **Step 2:** n ≈ 2000 × 0.44 = 880 matching tuples.
- **Step 3:** A6 (secondary index, range): each of 880 records may be on a different block → 880 random I/Os.
- **Step 4:** 880 random seeks at 4ms each = 3520ms vs linear scan at 24ms → **linear scan wins by 146×**.

**Cost Calculation:**

```
Estimation:
  RF(salary 40k–120k) ≈ 0.44
  n ≈ 2000 × 0.44 = 880 qualifying tuples

A6 — Secondary Index, Range (salary 40k–120k), HT=2:
  n = 880 records (each possibly on different block!)
  Cost = (HT + n) × (tT + tS)
       = (2 + 880) × (0.1 + 4)
       = 882 × 4.1 = 3,616.2 ms  ← disastrous!

A1 — Linear Scan:
  Cost = 200×0.1 + 4 = 24.0 ms  ← wins by 150×

Threshold: when does A6 beat A1?
  (2+n)×4.1 < 24.0
  n < 24.0/4.1 - 2 ≈ 3.85 → n ≤ 3 matching records
  So: secondary index only helps if ≤3 tuples match!
```

**Cost Comparison Table:**

| Algorithm | n (matching) | Cost (ms) | Use when |
|-----------|-------------|-----------|----------|
| A6 Secondary Index | 880 | 3,616.2 | n ≤ 3 only |
| A1 Linear Scan ★ | all scanned | **24.0 ★** | n > 3 (almost always) |

> **Key Insight:** Secondary indexes for range queries become useful **only when extremely few tuples qualify** (n ≤ ~3 for this file). For any non-trivial range, linear scan is dramatically faster due to random I/O costs of secondary indexes.

---

### Example 35: Composite Index Match vs No-Match

**SQL Query A (Index Matches):**
```sql
SELECT * FROM Student WHERE gpa = 3.8 AND age = 21;
```

**SQL Query B (Index Does NOT Match — missing prefix):**
```sql
SELECT * FROM Student WHERE age = 21;
-- Composite index on (gpa, age) does NOT match age alone!
```

**Initial RA (both):**
```
Query A: σ_{gpa=3.8 ∧ age=21}(Student)
Query B: σ_{age=21}(Student)
```

**Optimized RA:**
```
Query A: A8 — use composite B+tree index on (gpa, age), HT=3
Query B: A1 — linear scan (composite index requires prefix gpa first)
         OR A4 — secondary index on age alone (HT=2, if it exists)
```

**Optimization Steps (Query A):**

- **Step 1:** Composite index on (gpa, age) — both attributes present with equality → full composite key match.
- **Step 2:** SC combined = NR / (V(gpa)×V(age)) = 1000/(20×50) = 1 expected tuple.
- **Step 3:** Use composite index: traverse 3 levels, fetch 1 record.

**Optimization Steps (Query B):**

- **Step 1:** Only age=21 is given. Composite index on (gpa, age) requires the PREFIX (gpa first).
- **Step 2:** age=21 alone does NOT match the composite index — cannot use it.
- **Step 3:** Options: A1 linear scan, OR secondary index on age alone (HT=2) if it exists.
- **Step 4:** SC(age=21) = 20 → A4: (2+20)×4.1 = 90.2ms vs A1: 14ms → **A1 wins**.

**Cost Calculation:**

```
Query A — gpa=3.8 AND age=21:
  A8 Composite Index (gpa, age), HT=3:
    Expected SC ≈ 1 tuple
    Cost = (HT+1)×(tT+tS) = (3+1)×4.1 = 16.4 ms  ← best

  A7 Single index on gpa (HT=2), SC(gpa=3.8)=50:
    Cost = (2+50)×4.1 = 213.2 ms → apply age in memory → 213.2 ms

  A1 Linear Scan:
    Cost = 100×0.1+4 = 14.0 ms  ← almost as fast as A8 here!

Query B — age=21 only:
  A4 Secondary index on age (HT=2), SC=20:
    Cost = (2+20)×4.1 = 90.2 ms

  A1 Linear Scan:
    Cost = 14.0 ms  ← clear winner

Tree Index Match Rule Recap:
  Composite B+tree on <a, b, c>:
    ✅ Matches: a=v1                 (prefix)
    ✅ Matches: a=v1 AND b=v2       (prefix)
    ✅ Matches: a=v1 AND b>v2       (prefix with range on last)
    ❌ No match: b=v2               (missing prefix a)
    ❌ No match: b=v2 AND c=v3      (missing prefix a)
```

**Cost Comparison Table:**

| Query | Algorithm | Cost (ms) | |
|-------|-----------|-----------|--|
| A (gpa+age) | A1 Linear Scan | 14.0 ★ | Nearly ties composite index |
| A (gpa+age) | A8 Composite Index | 16.4 | Slightly slower (small file) |
| A (gpa+age) | A7 Single (gpa) | 213.2 | Poor choice |
| B (age only) | A1 Linear Scan ★ | **14.0 ★** | Composite index inapplicable |
| B (age only) | A4 age-only index | 90.2 | Expensive (20 random I/Os) |

> **Key Insight:** A composite B+tree index **requires the leftmost prefix** of the key to be specified. Missing the prefix attribute makes the composite index unusable — you must fall back to a single-attribute index or linear scan.

---

### Example 36: Hash Index — Exact Match Only

**SQL Query A (Hash Index Works):**
```sql
-- Assume hash index on (sid, cid) for Takes
SELECT * FROM Takes WHERE sid = 500 AND cid = 'CS101';
```

**SQL Query B (Hash Index Does NOT Work):**
```sql
SELECT * FROM Takes WHERE sid = 500;
-- Hash index on (sid, cid) requires BOTH attributes with equality
```

**Initial RA:**
```
Query A: σ_{sid=500 ∧ cid='CS101'}(Takes)
Query B: σ_{sid=500}(Takes)
```

**Optimized RA:**
```
Query A: Hash index on (sid, cid) — equality on ALL key attributes ✅
Query B: A1 Linear scan (hash index unusable) or separate index on sid
```

**Optimization Steps:**

- **Step 1 (Query A):** Hash index on (sid, cid) — both attributes with equality → PERFECT match. Hash function applied, jump directly to bucket.
- **Step 2 (Query A):** Expected result: 1 tuple (sid+cid is likely a key for Takes).
- **Step 3 (Query B):** Hash index requires equality on EVERY attribute. `cid` not specified → cannot hash properly → index unusable.
- **Step 4 (Query B):** Must use linear scan or a separate index on sid alone.

**Cost Calculation:**

```
Query A — sid=500 AND cid='CS101' via Hash Index:
  Cost ≈ 1×(tT + tS) + bucket_scan
       = 1×(0.1+4) + small overhead ≈ 4.1–8.2 ms
  (Essentially: one hash computation + one or two I/Os)

Query A — A1 Linear Scan:
  Cost = 200×0.1 + 4 = 24.0 ms

Query B — sid=500 only:
  Hash index INAPPLICABLE
  A1 Linear Scan: Cost = 24.0 ms
  If secondary index on sid existed (HT=3, SC(sid,Takes)=5):
    A4: (3+5)×4.1 = 32.8 ms  ← worse than linear scan!

Hash Index Match Rule:
  Hash index on <a, b, c>:
    ✅ Matches: a=v1 AND b=v2 AND c=v3  (all with equality)
    ❌ No match: a=v1 AND b=v2          (c missing)
    ❌ No match: a=v1                   (b,c missing)
    ❌ No match: a>v1 AND b=v2 AND c=v3 (range on a)
    ❌ No match: a=v1 AND b>v2 AND c=v3 (range on any)
```

**Cost Comparison Table:**

| Query | Algorithm | Cost (ms) | |
|-------|-----------|-----------|--|
| A (sid+cid) | Hash Index ★ | **~8.2 ★** | Direct bucket access |
| A (sid+cid) | A1 Linear | 24.0 | |
| B (sid only) | Hash Index | N/A | Unusable |
| B (sid only) | A1 Linear ★ | **24.0 ★** | Only option |

> **Key Insight:** Hash indexes are extremely fast for exact-match queries on the FULL key but are completely useless for partial key lookups, range queries, or any condition that isn't strict equality on every indexed attribute.

---

### Example 37: Selectivity Calculation with Histograms

**SQL Query:**
```sql
SELECT * FROM Employee WHERE salary BETWEEN 60000 AND 70000;
```

**Initial RA:**
```
σ_{salary≥60000 ∧ salary≤70000}(Employee)
```

**Optimized RA:**
```
Use histogram data if available; otherwise uniform distribution assumption.
A1 likely wins. Verify with RF.
```

**Optimization Steps:**

- **Step 1:** Without histogram: assume uniform distribution over salary range.
- **Step 2:** V(salary, Employee) = 500 distinct values. Assume range 20k–120k (width=100k).
- **Step 3:** RF = (70000-60000) / (120000-20000) = 10000/100000 = **0.10**.
- **Step 4:** n = NR × RF = 2000 × 0.10 = 200 tuples.
- **Step 5:** With histogram (say 15% of employees earn 60k–70k): n = 300 tuples.
- **Step 6:** Either way, n >> 3 → A1 wins over secondary index.

**Cost Calculation:**

```
Without histogram (uniform assumption):
  RF(salary 60k–70k) = (70k-60k)/(120k-20k) = 0.10
  n = 2000 × 0.10 = 200 tuples

With histogram (more accurate):
  Say histogram shows 60k–70k bucket = 15% of tuples
  n = 2000 × 0.15 = 300 tuples

A6 — Secondary Index (HT=2):
  Uniform: (2+200)×4.1 = 828.2 ms
  Histogram: (2+300)×4.1 = 1238.2 ms

A1 — Linear Scan:
  Cost = 200×0.1 + 4 = 24.0 ms  ← always wins here

Output block estimate:
  b_out = ceil(n / FR) = ceil(200/10) = 20 blocks (uniform)
           or ceil(300/10) = 30 blocks (histogram)
```

**Cost Comparison Table:**

| Assumption | n (tuples) | A6 Index (ms) | A1 Scan (ms) | Best |
|------------|-----------|---------------|-------------|------|
| Uniform RF=0.10 | 200 | 828.2 | **24.0 ★** | A1 |
| Histogram RF=0.15 | 300 | 1238.2 | **24.0 ★** | A1 |

> **Key Insight:** Histograms improve the **accuracy** of selectivity estimates but don't change the algorithmic choice here. Knowing the true RF matters most when the estimate is near the break-even point (n ≈ 3 for this case), which almost never happens for range queries on salary.

---

### Example 38: Conjunctive Selection — All Three Methods Compared (A7, A8, A9)

**SQL Query:**
```sql
SELECT * FROM Supply WHERE sup_id = 10 AND pid = 200;
-- Assume: secondary index on sup_id (HT=3), secondary index on pid (HT=3)
-- Also: composite index on (sup_id, pid), HT=4
```

**Initial RA:**
```
σ_{sup_id=10 ∧ pid=200}(Supply)
```

**Optimized RA:**
```
A8: Composite index on (sup_id, pid) — both equality conditions → best
```

**Optimization Steps:**

- **Step 1:** SC(sup_id=10, Supply) = ceil(10000/500) = 20 tuples.
- **Step 2:** SC(pid=200, Supply) = ceil(10000/800) = ceil(12.5) = 13 tuples.
- **Step 3:** Combined SC ≈ 10000/(500×800) ≈ 0.025 → at most 1 tuple.
- **Step 4:** A7 best: use pid index (SC=13 < 20). A8: composite. A9: intersect RIDs.

**Cost Calculation:**

```
A1 — Linear Scan:
  Cost = 400×0.1 + 4 = 44.0 ms

A7 — Best single index (pid, SC=13):
  Cost = (HT_pid + n) × (tT+tS)
       = (3 + 13) × 4.1 = 65.6 ms
  Apply sup_id=10 in memory on 13 records → free
  [worse than linear scan!]

A7 — sup_id index (SC=20):
  Cost = (3+20) × 4.1 = 94.3 ms  ← even worse

A9 — Intersection of record pointers:
  sup_id index scan: get 20 pointers → cost ≈ 3×4.1 = 12.3ms (index traverse)
  pid index scan:    get 13 pointers → cost ≈ 3×4.1 = 12.3ms
  Intersect in memory: find ≤1 common pointer → free
  Fetch 1 record: 1×4.1 = 4.1 ms
  Total ≈ 12.3 + 12.3 + 4.1 = 28.7 ms

A8 — Composite Index on (sup_id, pid), HT=4:
  Expected result: ≈1 tuple
  Cost = (HT+1)×(tT+tS) = (4+1)×4.1 = 20.5 ms  ← BEST

A1 comparison: 44.0ms
```

**Cost Comparison Table:**

| Algorithm | Key Feature | Cost (ms) | |
|-----------|------------|-----------|--|
| A1 Linear Scan | No index | 44.0 | |
| A7 (pid index) | SC=13 | 65.6 | ❌ Worse than scan |
| A9 Intersection | Both indexes | 28.7 | |
| A8 Composite Index ★ | (sup_id,pid) key | **20.5 ★** | Best |

> **Key Insight:** When a composite index covers **all** equality conditions in a conjunct, A8 outperforms both A7 and A9 because it traverses only ONE index path to the (likely unique) result. A9 requires traversing TWO index paths plus intersection overhead.

---

### Example 39: Conjunctive — Mixed Indexed and Non-Indexed Conditions

**SQL Query:**
```sql
SELECT * FROM Parts WHERE color = 'Red' AND weight > 10 AND pname LIKE 'Bolt%';
```

**Initial RA:**
```
σ_{color='Red' ∧ weight>10 ∧ pname LIKE 'Bolt%'}(Parts)
```

**Optimized RA:**
```
A7: Use secondary index on color (most selective indexed condition).
    Apply weight>10 and LIKE in memory. LIKE has no index anyway.
```

**Optimization Steps:**

- **Step 1:** pname LIKE 'Bolt%' — no index on pname → **cannot be used for index access**.
- **Step 2:** weight>10: no index on weight → cannot be used.
- **Step 3:** color='Red': secondary index exists (HT=2). SC(color,Parts) = ceil(800/8) = 100.
- **Step 4:** Only ONE condition has an index → only A7 is applicable (A8, A9 need multiple indexes).
- **Step 5:** Compare A7 (color index, n=100) vs A1.

**Cost Calculation:**

```
SC(color='Red', Parts) = ceil(800/8) = 100 tuples

A1 — Linear Scan:
  Cost = 48×0.1 + 4 = 4.8 + 4 = 8.8 ms

A7 — Secondary Index on color (HT=2), n=100:
  Cost = (HT + n) × (tT+tS)
       = (2 + 100) × 4.1 = 418.2 ms
  Then apply weight>10 and LIKE in memory on 100 records → free
  Total = 418.2 ms  ← MUCH worse!

A9/A8 — Not applicable (only one indexed condition)

Decision: USE A1 (linear scan, 8.8ms)
  color='Red' RF = 1/8 = 0.125 → 100/800 = 12.5% qualify
  When n=100 >> 3 (break-even), secondary index loses badly.
```

**Cost Comparison Table:**

| Algorithm | n | Cost (ms) | |
|-----------|---|-----------|--|
| A7 (color index) | 100 | 418.2 | ❌ |
| A1 Linear Scan ★ | 800 scanned | **8.8 ★** | Winner |

> **Key Insight:** When **no index exists for some conditions** (LIKE, unindexed columns), A9 is not applicable for those conditions. And if the one available index has poor selectivity (SC=100 out of 800), it's faster to scan the whole small file. Always check file size: BR=48 means even a full scan is very cheap here.

---

### Example 40: Group-By with Selection — Push Down and Aggregate

**SQL Query:**
```sql
SELECT dno, COUNT(*) FROM Employee WHERE age > 35 GROUP BY dno;
```

**Initial RA (extended):**
```
γ_{dno, COUNT(*)}(σ_{age>35}(Employee))
-- γ = generalized aggregate/group-by operator
```

**Optimized RA:**
```
γ_{dno, COUNT(*)}(σ_{age>35}(Employee))
-- σ pushed as early as possible; aggregate after
```

**Optimization Steps:**

- **Step 1:** Cannot push γ past σ generally (different semantics). Push σ first.
- **Step 2:** σ_{age>35}: RF ≈ (50-35)/50 × 2 ≈ 0.4 (assume ages 20–60, 40% above 35).
- **Step 3:** n ≈ 2000×0.4 = 800 tuples pass σ. Then group-by dno (V=10 groups).
- **Step 4:** Sort on dno (for grouping) or use hash aggregation.
- **Step 5:** σ has secondary index on age (HT=2). SC(age>35) ≈ 800 → A1 wins over index.

**Cost Calculation:**

```
Step 1: σ_{age>35}(Employee)
  RF ≈ 0.40, n ≈ 800 tuples, b_out = ceil(800/10) = 80 blocks
  A1 cost: 200×0.1+4 = 24.0 ms

Step 2: γ_{dno, COUNT(*)} — sort-based aggregation
  Sort on dno: BR_in=80, B=22 buffers
    Pass 0 runs: ceil(80/22)=4; Merge passes: ceil(log21(4))=1; Total passes=2
    Sort cost: 2×80×2-80 = 240 transfers = 24+8×4 = 56ms
  Final scan to aggregate: 80×0.1+4 = 12ms
  Group-by output: 10 rows (one per dno) → negligible

Hash-based aggregation (alternative):
  Partition 80 blocks by dno (10 buckets ≈ 8 blocks each) → fits in memory
  Cost ≈ 2×80×0.1 + seeks ≈ 16+20 = 36ms  (faster than sort-based)

Total (sort-based):  24.0 + 56 + 12 = 92 ms
Total (hash-based):  24.0 + 36     = 60 ms  ← better
```

**Cost Comparison Table:**

| Plan | σ Cost | Aggregation | Total (ms) | |
|------|--------|-------------|-----------|--|
| Scan + Sort-based GROUP BY | 24.0 | 68.0 | 92.0 | |
| Scan + Hash-based GROUP BY ★ | 24.0 | 36.0 | **60.0 ★** | |

> **Key Insight:** For GROUP BY, hash-based aggregation often beats sort-based when there are few distinct groups (here, 10 groups) because the hash table fits easily in memory. Sort-based is preferred when output must also be sorted by the grouping attribute.

---

## SECTION B — Join Algorithm Comparisons (Examples 41–52)

---

### Example 41: NLJ vs BNLJ — Detailed Buffer Analysis

**SQL Query:**
```sql
SELECT * FROM Supplier S, Supply SP WHERE S.sup_id = SP.sup_id;
```

**Initial RA:**
```
Supplier ⋈_{sup_id=sup_id} Supply
```

**Optimized RA:**
```
BNLJ with Supplier (BR=42) as outer, Supply (BR=400) as inner
-- Choose smaller relation as outer for both NLJ and BNLJ
```

**Optimization Steps:**

- **Step 1:** Supplier: BR=42, NR=500. Supply: BR=400, NR=10000.
- **Step 2:** Smaller relation = Supplier (BR=42) → outer loop.
- **Step 3:** For BNLJ with B=22 buffers: use B-2=20 pages for outer chunks, 1 for inner, 1 for output.
- **Step 4:** Outer chunks: ceil(42/20) = 3 chunks. Inner (Supply) scanned 3 times.

**Cost Calculation:**

```
NLJ — Supplier outer (NR=500), Supply inner:
  Worst case: NR_outer × B_inner + B_outer
            = 500 × 400 + 42 = 200,042 transfers
  Cost = 200042×0.1 + (500+1)×4 = 20004.2 + 2004 = 22,008 ms

NLJ — Supply outer (NR=10000), Supplier inner:
  Worst: 10000×42+400 = 420,400 → 42040+1604 = 43,644 ms  ← worse!

BNLJ — Supplier outer (BR=42), Supply inner (with B=22):
  Outer chunk size = B-2 = 20 pages per chunk
  Chunks of Supplier: ceil(42/20) = 3
  Supply scanned once per chunk: 3 × 400 = 1200 inner scans + 42 outer reads
  Total transfers = 42 + 3×400 = 1242
  Cost = 1242×0.1 + (3+3)×4 = 124.2 + 24 = 148.2 ms

BNLJ — Supply outer (BR=400), Supplier inner:
  Chunks: ceil(400/20) = 20 chunks
  Total transfers = 400 + 20×42 = 1240
  Cost = 1240×0.1 + (20+20)×4 = 124 + 160 = 284 ms  ← worse

Best BNLJ: Supplier outer = 148.2 ms

Best case (Supplier fits in memory, B≥44):
  Cost = B_Supplier + B_Supply = 42 + 400 = 442 transfers
       = 44.2 + 8 = 52.2 ms
```

**Cost Comparison Table:**

| Algorithm | Outer | Worst (ms) | B=22 (ms) | Best (ms) |
|-----------|-------|-----------|----------|----------|
| NLJ | Supplier | 22,008 | — | 52.2 |
| NLJ | Supply | 43,644 | — | 52.2 |
| BNLJ ★ | Supplier | 148.2 ★ | **148.2 ★** | 52.2 |
| BNLJ | Supply | 284.0 | 284.0 | 52.2 |

> **Key Insight:** With B=22 pages, BNLJ reduces NLJ's worst case from 22,008ms to **148ms** — a **148× speedup**. The key is that B-2=20 pages of the outer are loaded at once, so the inner (Supply=400 blocks) is scanned only 3 times instead of 500.

---

### Example 42: Indexed NLJ — When it Beats All Others

**SQL Query:**
```sql
SELECT * FROM Course C, Takes T WHERE C.cid = T.cid AND C.dept = 'CS';
```

**Initial RA:**
```
σ_{dept='CS'}(Course) ⋈_{cid=cid} Takes
```

**Optimized RA:**
```
σ_{dept='CS'} applied first → 20 Course tuples
INLJ: Course_CS outer, index on Takes.cid (assume secondary, HT=3)
```

**Optimization Steps:**

- **Step 1:** Push σ_{dept='CS'} to Course: 100/5 = 20 tuples → 2 blocks.
- **Step 2:** Index on Takes.cid (HT=3). SC(cid, Takes) = 50 per course.
- **Step 3:** INLJ: Course_CS outer (NR=20), probe Takes index for each Course tuple.
- **Step 4:** Each probe fetches SC(cid,Takes) = 50 Takes tuples. These are scattered → random I/Os.
- **Step 5:** Compare all join methods on this filtered data.

**Cost Calculation:**

```
After filter: Course_CS = 20 tuples ≈ 2 blocks

INLJ — Course_CS outer (NR=20), probe Takes.cid index (HT=3, SC=50):
  c = cost of one index probe on Takes.cid
    = (HT + SC) × (tT+tS) = (3+50) × 4.1 = 217.3 ms per probe  ← huge!
  Total = B_Course_CS + NR_Course_CS × c
        = 2×0.1 + 1×4 [outer scan] + 20 × 217.3
        = 4.2 + 4346 = 4,350 ms  ← terrible (50 random I/Os per probe!)

INLJ with CLUSTERING index on Takes.cid (SC=50 consecutive):
  c = (HT + ceil(SC/FR)) × (tT+tS) = (3+2) × 4.1 = 20.5 ms
  Total = 4.2 + 20×20.5 = 4.2 + 410 = 414.2 ms

Hash Join (Course_CS=2 blocks, Takes=200 blocks):
  3×(2+200) = 606 transfers = 606×0.1+10×4 = 100.6 ms  ← best

Sort-Merge Join:
  Sort Course_CS (2 blks): trivial ≈ 5ms
  Sort Takes (200 blks): ≈80ms
  Merge: 2+200=202 → 28.2ms
  Total ≈ 5+80+28.2 = 113.2 ms
```

**Cost Comparison Table:**

| Algorithm | Cost (ms) | Notes |
|-----------|-----------|-------|
| INLJ (secondary) | 4,350 | 50 random I/Os/probe = terrible |
| INLJ (clustering) | 414.2 | Better, still slow |
| Sort-Merge | 113.2 | Good |
| Hash Join ★ | **100.6 ★** | Best |

> **Key Insight:** INLJ is beneficial only when **each outer tuple has very few matching inner tuples** (SC ≈ 1). Here SC(cid,Takes)=50, so each Course probe retrieves 50 scattered Takes records. Hash join avoids all this random I/O by partitioning first.

---

### Example 43: Sort-Merge Join — Shared Sort Benefit

**SQL Query:**
```sql
SELECT * FROM Employee E, Department D
WHERE E.dno = D.dno
ORDER BY E.dno;
```

**Initial RA:**
```
SORT_{dno}(Employee ⋈_{dno=dno} Department)
```

**Optimized RA:**
```
(Sort Employee on dno) ⋈_{sort-merge} Department
Then pass sorted result directly to ORDER BY — no extra sort!
```

**Optimization Steps:**

- **Step 1:** Query requires ORDER BY dno — output must be sorted on dno.
- **Step 2:** Sort-Merge Join also requires sorting both inputs on dno.
- **Step 3:** **Key insight**: if we use SMJ, the Employee sort for the join serves double duty as the ORDER BY sort → save one sort pass.
- **Step 4:** Compare SMJ (with shared sort) vs Hash Join (which requires a separate sort for ORDER BY).

**Cost Calculation:**

```
Sort-Merge Join (Employee: BR=200, Dept: BR=3, B=22):
  Sort Employee on dno:
    Pass 0: ceil(200/22)=10 runs; Merge passes: ceil(log21(10))=1; Total=2 passes
    Cost: 2×200×2-200 = 600 transfers ≈ 60+10×4 = 100ms
  Sort Dept on dno:
    3 blocks → fits in memory, 1 pass, trivial ≈ 4.3ms
  Merge: 200+3=203 transfers ≈ 20.3+2×4 = 28.3ms
  ORDER BY: FREE! (output already sorted on dno from SMJ)
  TOTAL: 100 + 4.3 + 28.3 + 0 = 132.6 ms

Hash Join (Employee: 200, Dept: 3):
  Hash join: 3×(200+3)=609 transfers ≈ 90ms
  Sort for ORDER BY: need to sort join result
    Join result size: NR_Employee (each emp matches 1 dept) = 2000 tuples ≈ 200 blocks
    Sort cost: 2×200×2-200 = 600 transfers ≈ 100ms
  TOTAL: 90 + 100 = 190 ms
```

**Cost Comparison Table:**

| Plan | Join Cost | ORDER BY Cost | Total (ms) | |
|------|-----------|---------------|-----------|--|
| Hash Join + Sort ★ baseline | 90 | 100 | 190 | |
| Sort-Merge Join ★ (shared sort) | 132.6 | **0 (free!)** | **132.6 ★** | |
| BNLJ + Sort | 76.3 | 100 | 176.3 | |

> **Key Insight:** When a query requires both a JOIN and an ORDER BY on the same attribute, **Sort-Merge Join eliminates the separate ORDER BY sort**. This "shared sort" optimization can save one full sort pass — significant for large relations.

---

### Example 44: Hash Join — Partition Overflow

**SQL Query:**
```sql
SELECT * FROM Student S, Takes T WHERE S.sid = T.sid;
-- B = 6 buffer pages only (very limited memory)
```

**Initial RA:**
```
Student ⋈_{sid=sid} Takes
```

**Optimized RA:**
```
Hash join with recursive partitioning if needed
```

**Optimization Steps:**

- **Step 1:** B=6 buffers. Phase 1 creates B-1=5 partitions.
- **Step 2:** Student: 100 blocks → each partition ≈ 100/5 = 20 blocks.
- **Step 3:** Phase 2: For each partition, build hash table on Student partition.
  - Student partition: 20 blocks. Need 20 + 2 buffers (1 Takes input, 1 output) = 22 pages.
  - Available: B=6. **OVERFLOW!** 20 > B-2=4 pages for in-memory hash table.
- **Step 4:** Must recurse: re-partition the overflowed partition using a different hash function.
- **Step 5:** Alternative: use sort-merge join which needs only B=3 buffers minimum.

**Cost Calculation:**

```
Hash Join with B=6 (limited memory):
  Phase 1 — Partition:
    Read+write both: 2×(100+200) = 600 transfers
    5 partitions created.

  Partition sizes: Student ≈ 20 blk, Takes ≈ 40 blk each

  Phase 2 check: can Student partition (20 blk) fit in B-2=4 pages? NO!
    → OVERFLOW: need recursive partitioning

  Recursive partition of each Student partition (20 blk into 5 sub-partitions = 4 blk):
    4 blk fits in 4 pages (B-2=4) ✓
    Extra I/O: 2×5×(20+40) = 600 more transfers

  Total hash join with recursive partitioning:
    ≈ 600 + 600 + 300 (final probe) = 1500 transfers ≈ 195ms

Sort-Merge Join (also with B=6):
  Sort Student (BR=100, B=6):
    Pass 0: ceil(100/6)=17 runs; Merge: ceil(log5(17))=2 passes; Total=3 passes
    Cost: 2×100×3-100 = 500 transfers ≈ 70ms
  Sort Takes (BR=200, B=6):
    Pass 0: ceil(200/6)=34; Merge: ceil(log5(34))=3; Total=4 passes
    Cost: 2×200×4-200 = 1400 → 180ms
  Merge: 300 transfers ≈ 38ms
  Total: 70+180+38 = 288ms

BNLJ (B=6, chunk=4 pages):
  Outer chunks: ceil(100/4)=25; Total = 100+25×200=5100 transfers ≈ 610ms
```

**Cost Comparison Table:**

| Algorithm | B=6 Cost (ms) | B=22 Cost (ms) | Notes |
|-----------|-------------|---------------|-------|
| Hash Join (no overflow) | — | 130 | Needs B≥22 approx |
| Hash Join (recursive partition) | **195** | — | Extra passes |
| Sort-Merge ★ | **288** | 168 | More passes with small B |
| BNLJ | 610 | 148 | Degrades badly |

> **Key Insight:** Hash join degrades gracefully with recursive partitioning but becomes expensive. Sort-Merge Join is more **memory-efficient** (needs only B=3 minimum for basic operation) and degrades more predictably as B decreases.

---

### Example 45: Self-Join

**SQL Query:**
```sql
SELECT E1.ename, E2.ename AS manager
FROM Employee E1, Employee E2
WHERE E1.dno = E2.dno AND E1.eid <> E2.eid;
-- Find all employee pairs in the same department
```

**Initial RA:**
```
σ_{E1.dno=E2.dno ∧ E1.eid≠E2.eid}(Employee × Employee)
```

**Optimized RA:**
```
ρ_{E1}(Employee) ⋈_{E1.dno=E2.dno} ρ_{E2}(Employee)
  then filter E1.eid ≠ E2.eid on-the-fly
```

**Optimization Steps:**

- **Step 1:** Self-join: both inputs are the same Employee relation.
- **Step 2:** One copy acts as outer (E1), same table scanned as inner (E2).
- **Step 3:** Cannot use the same index for both sides simultaneously in standard NLJ.
- **Step 4:** Estimated output: for each of NR=2000 employees, avg SC(dno)=200 peers → 200×2000 = 400,000 pairs minus the n (eid≠eid) diagonal. Large intermediate!
- **Step 5:** Hash join on dno is best — partition Employee once, join within each bucket.

**Cost Calculation:**

```
Estimated output tuples:
  For dno group of size 200: pairs = 200×199 = 39,800 per group
  10 groups → total ≈ 398,000 pairs  (very large result!)

NLJ — E1 outer, E2 inner:
  Worst: NR_E1 × BR_E2 + BR_E1 = 2000×200+200 = 400,200 transfers
  Cost ≈ 40020 + 8008 ≈ 48,028 ms

BNLJ (B=22, chunk=20):
  Chunks: ceil(200/20)=10
  Total: 200 + 10×200 = 2200 transfers ≈ 310ms

Hash Join on dno:
  Only ONE copy of Employee to read twice (once per phase):
  Phase 1: Read Employee 200 blocks + write partitions 200 blocks = 400 transfers
  Phase 2: Read all partitions = 200 blocks
  Total = 400 + 200 = 600 transfers ≈ 90ms + overhead
  E1.eid ≠ E2.eid filter applied on-the-fly — free
```

**Cost Comparison Table:**

| Algorithm | Transfers | Cost (ms) | |
|-----------|----------|-----------|--|
| NLJ | 400,200 | 48,028 | ❌ |
| BNLJ | 2,200 | 310 | |
| Hash Join ★ | 600 | **~90 ★** | |

> **Key Insight:** Self-joins use the same physical table for both inputs. Hash join is particularly efficient here because the table is only physically read once per phase — the partitioning handles the logical split. The ≠ condition is applied on-the-fly during the probe phase.

---

### Example 46: Semi-Join and Exists

**SQL Query:**
```sql
SELECT S.sname FROM Student S
WHERE EXISTS (
    SELECT 1 FROM Takes T WHERE T.sid = S.sid AND T.cid = 'CS101'
);
```

**Initial RA:**
```
π_{sname}(Student ⋈_{sid ∈ π_{sid}(σ_{cid='CS101'}(Takes))} Student)
-- Equivalent to: Student ⋈_{sid} (π_{sid}(σ_{cid='CS101'}(Takes)))
```

**Optimized RA:**
```
π_{sname}(Student ⋈_{sid} (π_{sid}(σ_{cid='CS101'}(Takes))))
-- CORRELATED query → convert to semi-join
-- Inner block evaluated ONCE (cid='CS101' is independent of outer)
```

**Optimization Steps:**

- **Step 1:** EXISTS with inner query referencing outer tuple's sid → correlated.
- **Step 2:** BUT: the inner condition T.cid='CS101' is fixed → inner can be evaluated ONCE.
- **Step 3:** Execute inner: σ_{cid='CS101'}(Takes) → 50 Takes tuples → project to sid → at most 50 distinct sids.
- **Step 4:** Semi-join: find Students whose sid is in this set of 50 sids.
- **Step 5:** Build hash table of 50 sids (fits easily in memory), probe with Student scan.

**Cost Calculation:**

```
Step 1: σ_{cid='CS101'}(Takes) → linear scan
  Cost: 200×0.1+4 = 24ms → 50 tuples, ≤50 distinct sids

Step 2: π_{sid} on 50 tuples → ≤50 distinct sids → build in-memory hash set (~free)

Step 3: Scan Student (100 blocks), check each sid against hash set → semi-join
  Cost: 100×0.1+4 = 14ms
  Output: students whose sid is in the 50-sid set
         Expected: 50 distinct sids in Takes, 50 matching Student tuples

Total: 24 + 0 + 14 = 38 ms

Alternative — Correlated execution (naive):
  For each of 1000 Student tuples, run inner query:
    σ_{cid='CS101' ∧ sid=outer.sid}(Takes) → scan Takes or use index
    Without index: 1000 × (200×0.1+4) = 1000×24 = 24,000 ms  ← terrible!
    With secondary index on Takes.cid (HT=3):
      Each probe: (3+50)×4.1 = 217ms → but only 1 match needed → stop early
      ≈ 500ms average per probe → 500,000 ms  ← catastrophic!
```

**Cost Comparison Table:**

| Plan | Cost (ms) | Notes |
|------|-----------|-------|
| Correlated (no index) | 24,000 | Naive, terrible |
| Correlated (with index) | ~500,000 | Even worse! |
| Semi-join (de-correlated) ★ | **38 ★** | Evaluate inner once |

> **Key Insight:** **De-correlating** subqueries is one of the most impactful optimizations. Converting a correlated EXISTS subquery to a semi-join reduces cost from O(N×inner_cost) to O(inner_cost + outer_scan). Always check if the inner query can be evaluated once.

---

### Example 47: Anti-Join (NOT EXISTS / NOT IN)

**SQL Query:**
```sql
SELECT S.sname FROM Student S
WHERE S.sid NOT IN (SELECT T.sid FROM Takes T);
-- Find students who have never taken any course
```

**Initial RA:**
```
π_{sname}(Student) − π_{sname}(Student ⋈_{sid} Takes)
-- Equivalent to: π_{sname}(Student ⋈_{anti} Takes)
```

**Optimized RA:**
```
Anti-join: build hash set of all sids in Takes, scan Student,
           output those NOT in the hash set
```

**Optimization Steps:**

- **Step 1:** NOT IN equivalent to anti-join: Student tuples with no match in Takes.
- **Step 2:** V(sid,Takes)=1000 distinct sids → build in-memory hash set (1000 entries, tiny).
- **Step 3:** Scan Takes once to build set: BR_Takes = 200 blocks.
- **Step 4:** Scan Student, output tuples where sid ∉ hash set: BR_Student = 100 blocks.
- **Step 5:** Total = scan Takes + scan Student = 300 block reads.

**Cost Calculation:**

```
Hash-based Anti-Join:
  Step 1: Scan Takes, build hash set of all sid values
    Cost: 200×0.1+4 = 24ms → 1000 distinct sids in-memory hash (fits easily)

  Step 2: Scan Student, for each tuple check sid ∉ hash set
    Cost: 100×0.1+4 = 14ms
    Output: students NOT in Takes (expect some, say 10%)

  Total: 24 + 14 = 38 ms

Naive set-difference approach:
  π_{sname}(Student) − π_{sname}(Student ⋈ Takes)
  Step 1: Hash Join Student⋈Takes: 3×(100+200)=900 → 130ms
  Step 2: Project to sname: sort+dedup on ≤5000 result → expensive
  Step 3: Minus original Student: sort+dedup → more expensive
  Total >> 300ms

NULL handling note for NOT IN:
  If any sid in Takes is NULL, NOT IN returns 0 rows (SQL semantics!)
  NOT EXISTS handles NULLs correctly → prefer NOT EXISTS over NOT IN
```

**Cost Comparison Table:**

| Algorithm | Cost (ms) | Handles NULLs? |
|-----------|-----------|---------------|
| NOT IN naive (join+difference) | >300 | Tricky |
| Anti-join (hash set) ★ | **38 ★** | With care |
| NOT EXISTS (semi-join) | ~38 | Yes ✓ |

> **Key Insight:** Anti-joins are efficiently implemented by building a hash set of the inner relation's join attribute and scanning the outer relation, outputting non-matches. This achieves O(BR_inner + BR_outer) cost — the theoretical minimum.

---

### Example 48: Outer Join

**SQL Query:**
```sql
SELECT S.sname, T.cid FROM Student S LEFT OUTER JOIN Takes T ON S.sid = T.sid;
-- Include all students even if they have no courses
```

**Initial RA:**
```
Student ⟕_{sid=sid} Takes   -- Left Outer Join
```

**Optimized RA:**
```
Hash-based LOJ:
  Phase 1: Partition both on sid
  Phase 2: For each partition, join + emit unmatched Student tuples
```

**Optimization Steps:**

- **Step 1:** Left outer join: all Student tuples appear in result (with NULLs if no Takes match).
- **Step 2:** Cannot simply swap inner/outer relations (LOJ is asymmetric).
- **Step 3:** Hash join extended for LOJ: during probe phase, track which outer (Student) tuples had matches; emit unmatched ones with NULL Takes fields.
- **Step 4:** Result size = NR_Takes (for matching) + (NR_Student − matched students).

**Cost Calculation:**

```
Result size estimation:
  Matching students: min(NR_Student, V(sid,Takes)) = min(1000,1000) = 1000
  All students appear: 1000 rows with Takes data + (NR_Student - 1000) with NULLs
  Assuming all 1000 students appear in Takes:
    Result = 5000 rows (one per Takes tuple, student data replicated)

Hash LOJ Cost (same as regular hash join):
  3×(BR_Student + BR_Takes) = 3×(100+200) = 900 transfers
  Cost = 900×0.1 + seeks ≈ 90+40 = 130ms
  (Extra tracking of unmatched Student tuples is in-memory → free)

Sort-Merge LOJ:
  Sort Student (100 blk): 2 passes → 300 transfers ≈ 50ms
  Sort Takes (200 blk): 2 passes → 600 transfers ≈ 80ms
  Merge with NULL-tracking: 300 transfers ≈ 38ms
  Total ≈ 168ms
```

**Cost Comparison Table:**

| Algorithm | Cost (ms) | Notes |
|-----------|-----------|-------|
| Hash LOJ ★ | **130 ★** | Same cost as regular hash join |
| Sort-Merge LOJ | 168 | Sorted output by sid |
| BNLJ LOJ | 2414 | Track unmatched outer tuples |

> **Key Insight:** Outer joins have the **same algorithmic cost** as their inner join counterparts — the only difference is bookkeeping for unmatched tuples, which is done in-memory. Choose the same algorithm you would for an inner join.

---

### Example 49: Three-Way Join — All Possible Left-Deep Orderings

**SQL Query:**
```sql
SELECT S.sname, C.cname, T.grade
FROM Student S, Takes T, Course C
WHERE S.sid=T.sid AND T.cid=C.cid;
```

**Initial RA:**
```
Student ⋈_{sid} Takes ⋈_{cid} Course
```

**Optimized RA:**
```
Evaluate all 6 left-deep orderings; choose minimum cost.
Best: (Course ⋈_{cid} Takes) ⋈_{sid} Student
```

**Optimization Steps:**

- **Step 1:** Three relations: Student(BR=100), Takes(BR=200), Course(BR=9).
- **Step 2:** Three possible "first join" choices: (S⋈T), (T⋈C), (S⋈C — needs push for cid).
- **Step 3:** Six left-deep orderings total (3! = 6).
- **Step 4:** Estimate intermediate result sizes for each ordering.
- **Step 5:** Estimate total cost using hash joins throughout.

**Cost Calculation:**

```
Ordering 1: (Student ⋈_{sid} Takes) ⋈_{cid} Course
  Join 1: S⋈T: result = NR_Takes = 5000 tuples ≈ 5000/25 = 200 blocks
  Join 2: (200 blk) ⋈ Course(9 blk): 3×(200+9) = 627 → 90ms
  Total: 3×(100+200) + 90 = 900+90 = 990 → 130+90 = 220ms

Ordering 2: (Course ⋈_{cid} Takes) ⋈_{sid} Student
  Join 1: C⋈T: result = NR_Takes = 5000 tuples ≈ 200 blocks
  Same total: 3×(9+200) + 3×(200+100) = 627+900 = 1527 → ~190ms

Ordering 3: (Student ⋈_{sid} Takes) then ⋈ Course [same as 1]
  = 220ms

Ordering 4: (Course ⋈_{cid} Takes) ⋈_{sid} Student [same as 2]
  = 190ms

Ordering 5: Takes ⋈_{sid} Student → then ⋈_{cid} Course
  Join 1: T⋈S: 3×(200+100) = 900 → 130ms; result = 5000 tuples = 200 blk
  Join 2: (200) ⋈ Course(9): 3×(200+9) = 627 → 90ms
  Total = 130+90 = 220ms

Ordering 6: Takes ⋈_{cid} Course → then ⋈_{sid} Student
  Join 1: T⋈C: 3×(200+9) = 627 → 90ms; result = 5000 tuples = 200 blk
  Join 2: (200) ⋈ Student(100): 3×(200+100) = 900 → 130ms
  Total = 90+130 = 220ms

Summary: Most orderings ≈ 220ms because Takes(BR=200) dominates.
  The smallest "left" input doesn't matter much here — Takes is always largest.
```

**Cost Comparison Table:**

| Ordering | Join 1 Cost | Intermediate | Join 2 Cost | Total (ms) |
|----------|------------|-------------|-------------|-----------|
| (S⋈T)⋈C | 130 | 200 blk | 90 | 220 |
| (C⋈T)⋈S | 90 | 200 blk | 130 | **220** |
| (T⋈S)⋈C | 130 | 200 blk | 90 | 220 |
| (T⋈C)⋈S ★ | 90 | 200 blk | 130 | 220 |
| (S⋈C)? | N/A | — | — | N/A (no join cond) |

> **Key Insight:** When one relation (Takes) is much larger and must appear in the first join regardless of ordering, all left-deep orderings have similar cost. Selection pushdown is more impactful than join ordering in such cases — if σ reduces Takes significantly, the ordering matters much more.

---

### Example 50: Join with Projection Pushdown — Reducing Record Width

**SQL Query:**
```sql
SELECT S.sname FROM Student S, Takes T, Course C
WHERE S.sid=T.sid AND T.cid=C.cid AND C.dept='CS';
```

**Initial RA (after σ pushdown):**
```
π_{sname}(Student ⋈_{sid} Takes ⋈_{cid} σ_{dept='CS'}(Course))
```

**Optimized RA (with π pushdown):**
```
π_{sname}(
  π_{sname,sid}(Student)
  ⋈_{sid}
  π_{sid,cid}(Takes)
  ⋈_{cid}
  π_{cid}(σ_{dept='CS'}(Course))
)
```

**Optimization Steps:**

- **Step 1:** σ_{dept='CS'} pushed to Course → 20 tuples.
- **Step 2:** Push π down: Course only needs cid after σ. Takes only needs sid, cid. Student only needs sname, sid.
- **Step 3:** Smaller records → higher FR → fewer blocks → cheaper joins.
- **Step 4:** Calculate new FR and BR after projection.

**Cost Calculation:**

```
Without π pushdown:
  Student: LR=50B, FR=10, BR=100
  Takes: LR=20B, FR=25, BR=200

With π pushdown:
  π_{sname,sid}(Student): LR≈20B, FR=25, BR=ceil(1000/25)=40  [saved 60 blocks!]
  π_{sid,cid}(Takes): LR≈8B, FR=64, BR=ceil(5000/64)=79  [saved 121 blocks!]
  π_{cid}(Course_CS=20 tuples): LR≈4B, trivial

Join costs WITHOUT π pushdown:
  σ(Course)=2 blk ⋈ Takes=200 blk: 3×202 = 606 → 90ms
  Intermediate = 5000 tuples × (20+40)B = 1200 blk
  Intermediate ⋈ Student=100 blk: 3×(1200+100) = 3900 → 470ms
  Total ≈ 90+470 = 560ms

Join costs WITH π pushdown:
  Course_CS(cid only) ⋈ π(Takes)(2 blk ⋈ 79 blk): 3×81 = 243 → 50ms
  Intermediate = 1000 tuples × (cid+sid) ≈ 16B → 1000/64 = 16 blk
  Intermediate(16) ⋈ π(Student)(40 blk): 3×56 = 168 → 42ms
  Total ≈ 50+42 = 92ms  (vs 560ms — 6× speedup!)
```

**Cost Comparison Table:**

| Plan | BR_Takes | Intermediate | Total (ms) | |
|------|---------|-------------|-----------|--|
| No projection pushdown | 200 blk | 1200 blk | 560 | |
| With projection pushdown ★ | 79 blk | 16 blk | **92 ★** | 6× faster! |

> **Key Insight:** Projection pushdown **reduces record width early**, which reduces blocking factors and thus the number of blocks for intermediate results. Here, pushing π reduces the intermediate result from 1200 blocks to just 16 blocks — a 75× reduction that translates to a 6× speedup.

---

## SECTION C — Full Query Optimization Workflows (Examples 51–65)

---

### Example 51: Subquery Flattening — Uncorrelated to Join

**SQL Query:**
```sql
SELECT sname FROM Student
WHERE sid IN (SELECT sid FROM Takes WHERE cid = 'CS101');
```

**Initial RA (naive):**
```
For each Student tuple:
  Evaluate σ_{cid='CS101'}(Takes) → check if sid is in result
  [correlated execution: 1000 × full scan of Takes]
```

**Optimized RA (flattened to join):**
```
π_{sname}(Student ⋈_{sid} σ_{cid='CS101'}(Takes))
```

**Optimization Steps:**

- **Step 1:** Detect that IN subquery is uncorrelated (inner doesn't reference outer's tuple — only outer's sid column matters for the IN test).
- **Step 2:** **Subquery flattening**: rewrite as join. σ_{cid='CS101'}(Takes) evaluated once.
- **Step 3:** Execute inner block: σ → 50 tuples. Join with Student.
- **Step 4:** Apply π_{sname} on-the-fly.

**Cost Calculation:**

```
Naive correlated evaluation:
  For each of 1000 Student tuples:
    Scan Takes (200 blocks), check cid='CS101'
  Cost = 1000 × (200×0.1+4) = 1000×24 = 24,000 ms

Flattened plan:
  σ_{cid='CS101'}(Takes): 200×0.1+4 = 24ms → 50 tuples ≈ 2 blocks
  Hash join Student(100 blk) ⋈ σ_result(2 blk):
    3×(100+2) = 306 transfers ≈ 306×0.1+10×4 = 70.6ms
  π_{sname} on-the-fly: free
  TOTAL: 24 + 70.6 = 94.6 ms
  Speedup: 24000/94.6 ≈ 254× faster!
```

**Cost Comparison Table:**

| Plan | Strategy | Cost (ms) | Speedup |
|------|----------|-----------|---------|
| Correlated IN | 1000 × scan | 24,000 | 1× |
| Flattened to join ★ | eval once + join | **94.6 ★** | **254×** |

> **Key Insight:** Flattening correlated/uncorrelated subqueries into joins is often the **highest-impact single optimization**. Converting O(N × inner_cost) to O(inner_cost + join_cost) provides multiplicative speedups.

---

### Example 52: Scalar Subquery in WHERE Clause

**SQL Query:**
```sql
SELECT ename FROM Employee
WHERE salary > (SELECT AVG(salary) FROM Employee WHERE dno = 3);
```

**Initial RA:**
```
Block 1: σ_{salary > C}(Employee)   where C = AVG(salary | dno=3)
Block 2: γ_{AVG(salary)}(σ_{dno=3}(Employee))
```

**Optimized RA:**
```
Evaluate Block 2 FIRST (once), substitute constant C, then evaluate Block 1
```

**Optimization Steps:**

- **Step 1:** Two query blocks. Inner block (AVG) is uncorrelated — compute ONCE.
- **Step 2:** Block 2: σ_{dno=3}(Employee) → 200 tuples → compute AVG salary → constant C.
- **Step 3:** Block 1: σ_{salary>C}(Employee) → filter with constant → linear scan.
- **Step 4:** No join needed — scalar result.

**Cost Calculation:**

```
Block 2 — Compute AVG(salary) for dno=3:
  σ_{dno=3}: A1 linear scan = 24ms → 200 tuples
  AVG computed in-memory as tuples arrive → free (on-the-fly aggregation)
  Block 2 total: 24ms → returns constant C (e.g., C=65000)

Block 1 — σ_{salary>C}(Employee):
  RF(salary>65000) ≈ 0.5 (50% earn above avg)
  n ≈ 1000 qualifying tuples
  A1: 200×0.1+4 = 24ms (linear scan better than secondary index: n=1000 >> 3)
  π_{ename} on-the-fly: free

Total: 24 + 24 = 48 ms

Naive (re-evaluate inner for each outer tuple):
  1 outer scan + 2000×(scan for AVG) = 24 + 2000×24 = 48,024 ms
```

**Cost Comparison Table:**

| Strategy | Cost (ms) | |
|----------|-----------|--|
| Naive (re-evaluate per tuple) | 48,024 | ❌ |
| Evaluate inner once ★ | **48 ★** | 1000× speedup |

> **Key Insight:** Scalar subqueries (returning a single value) should **always be evaluated once** before the outer query. The result substitutes as a constant — this eliminates all repeated inner evaluations.

---

### Example 53: UNION Query Plan

**SQL Query:**
```sql
SELECT sname FROM Student WHERE gpa > 3.5
UNION
SELECT sname FROM Student WHERE age < 20;
```

**Initial RA:**
```
π_{sname}(σ_{gpa>3.5}(Student)) ∪ π_{sname}(σ_{age<20}(Student))
```

**Optimized RA:**
```
Option A: Scan Student ONCE, apply (gpa>3.5 OR age<20), project, sort+dedup
Option B: Separate plans per branch, union = sort+dedup
```

**Optimization Steps:**

- **Step 1:** UNION requires duplicate elimination (UNION ALL does not).
- **Step 2:** Both branches operate on the same table (Student) → scan once.
- **Step 3:** Rewrite as: `π_{sname}(σ_{gpa>3.5 ∨ age<20}(Student))` — single scan, OR condition.
- **Step 4:** Duplicate elimination via sort or hashing on sname.
- **Step 5:** Compare: one-scan vs two-scan approach.

**Cost Calculation:**

```
Two-scan approach:
  Scan 1: σ_{gpa>3.5}: 100×0.1+4 = 14ms → 250 tuples (25% qualify)
  Scan 2: σ_{age<20}:  100×0.1+4 = 14ms → 20 tuples (2% qualify)
  Combine: ≤270 tuples → sort for UNION dedup
  Sort (270 tuples ≈ 27 blk): in-memory (fits) ≈ 4ms
  Total: 14+14+4 = 32ms

One-scan approach (optimized):
  Scan Student ONCE, apply (gpa>3.5 OR age<20): 14ms
  Combined qualifying: ≤270 tuples
  Sort+dedup: 4ms
  Total: 14+4 = 18ms  ← BETTER (saves one full scan)

UNION ALL (no dedup needed):
  One-scan: just 14ms → output 270 tuples with possible duplicates
  Even cheaper!
```

**Cost Comparison Table:**

| Strategy | Scans | Cost (ms) | |
|----------|-------|-----------|--|
| Two-scan UNION | 2 | 32 | |
| One-scan UNION (OR rewrite) ★ | 1 | **18 ★** | |
| UNION ALL (no dedup) | 1 | 14 | If duplicates acceptable |

> **Key Insight:** When UNION branches operate on the **same table**, rewrite as a single OR-condition scan. This halves the I/O cost. Also: **UNION ALL is always cheaper than UNION** — avoid DISTINCT/dedup when semantics allow it.

---

### Example 54: INTERSECT Query Plan

**SQL Query:**
```sql
SELECT sid FROM Student WHERE gpa > 3.5
INTERSECT
SELECT sid FROM Takes WHERE cid = 'CS101';
```

**Initial RA:**
```
π_{sid}(σ_{gpa>3.5}(Student)) ∩ π_{sid}(σ_{cid='CS101'}(Takes))
```

**Optimized RA:**
```
Semi-join rewrite: π_{sid}(σ_{gpa>3.5}(Student) ⋈_{sid} σ_{cid='CS101'}(Takes))
-- Equivalent and more efficient
```

**Optimization Steps:**

- **Step 1:** INTERSECT ≡ semi-join when both sides project the same attribute.
- **Step 2:** Convert: Student sids that ALSO appear in CS101 Takes sids.
- **Step 3:** Inner: σ_{cid='CS101'}(Takes) → 50 tuples → build hash set of 50 sids.
- **Step 4:** Scan σ_{gpa>3.5}(Student) → probe hash set → output matching sids.
- **Step 5:** No separate sort/dedup needed (each Student sid is unique).

**Cost Calculation:**

```
Sort-Merge INTERSECT (naive):
  Sort branch 1 (sids from filtered Student): 250 tuples → trivial ≈ 5ms
  Sort branch 2 (sids from CS101 Takes): 50 tuples → trivial ≈ 2ms
  Merge-intersect scan: negligible
  Plus scan costs: 14+24 = 38ms
  Total: 38+5+2 = 45ms

Semi-join approach (optimized):
  σ_{cid='CS101'}(Takes): 24ms → 50 sids in memory hash set
  σ_{gpa>3.5}(Student): 14ms → probe hash set per tuple → output matches
  Total: 24+14 = 38ms  (no sort needed! Student.sid is unique)
```

**Cost Comparison Table:**

| Strategy | Cost (ms) | |
|----------|-----------|--|
| Sort-Merge INTERSECT | 45 | Standard approach |
| Semi-join rewrite ★ | **38 ★** | Avoids sort |

> **Key Insight:** INTERSECT on a key attribute can be rewritten as a semi-join, eliminating the need for sorting/deduplication because the key guarantees uniqueness. This saves the sort phase when one branch is small enough to build a hash set.

---

### Example 55: Aggregation Pushdown

**SQL Query:**
```sql
SELECT dno, SUM(salary) FROM Employee GROUP BY dno HAVING SUM(salary) > 500000;
```

**Initial RA:**
```
σ_{SUM(salary)>500000}(γ_{dno, SUM(salary)}(Employee))
```

**Optimized RA:**
```
σ_{SUM(salary)>500000}(γ_{dno, SUM(salary)}(Employee))
-- HAVING filter applied AFTER aggregation (cannot push before γ)
-- But: can we use an index to avoid scanning all employees?
```

**Optimization Steps:**

- **Step 1:** HAVING condition applies to aggregate result → cannot push before GROUP BY.
- **Step 2:** σ_{salary>X} cannot be pushed past γ (would change semantics).
- **Step 3:** Only option: scan all Employee tuples, group by dno, compute SUM, then filter groups.
- **Step 4:** Choose aggregation method: sort-based or hash-based.
- **Step 5:** Hash-based grouping: 10 groups (V(dno)=10) → tiny hash table.

**Cost Calculation:**

```
Sort-based GROUP BY:
  Sort Employee on dno (BR=200, B=22): 2 passes → cost 600 transfers ≈ 80ms
  Scan sorted result, compute SUM per group: 200×0.1+4 = 24ms
  Apply HAVING filter: free (only 10 group results)
  Total: 80+24 = 104ms

Hash-based GROUP BY (preferred for few groups):
  Scan Employee: 24ms
  As each tuple arrives, add salary to dno's running sum (10-entry hash table) → free
  After scan: apply HAVING → output groups with SUM>500k
  Total: 24ms  ← much better!

Note: No pre-filtering possible (HAVING is post-aggregate)
      No index helps (must touch all tuples for SUM)
```

**Cost Comparison Table:**

| Method | Cost (ms) | Notes |
|--------|-----------|-------|
| Sort-based GROUP BY | 104 | Useful if ORDER BY dno also needed |
| Hash-based GROUP BY ★ | **24 ★** | Much faster for few groups |

> **Key Insight:** **Hash-based aggregation** is almost always better than sort-based when the number of groups is small (V(dno)=10 here). The running-aggregate technique processes each tuple exactly once: one scan, zero materializations.

---

### Example 56: ORDER BY and Top-K Optimization

**SQL Query:**
```sql
SELECT sname, gpa FROM Student ORDER BY gpa DESC LIMIT 5;
-- Return top 5 students by GPA
```

**Initial RA:**
```
π_{sname,gpa}(SORT_{gpa DESC}(Student))
```

**Optimized RA:**
```
Option A: Full sort, output top 5
Option B: Sequential scan with in-memory priority queue (heap) of size 5
Option C: Secondary B+tree index on gpa — scan from highest to lowest, stop after 5
```

**Optimization Steps:**

- **Step 1:** LIMIT 5 → only need the 5 largest gpa values.
- **Step 2:** Full external sort: costly (reads/writes entire file) just for 5 results.
- **Step 3:** **Priority queue approach**: maintain a min-heap of size 5. Scan all tuples; replace min if new tuple > min. After scan: output heap.
- **Step 4:** **Index scan (B+tree on gpa)**: scan index from right (largest) to left. Collect 5 leaf entries. Fetch 5 records. Total: ~5 I/Os after traversal.

**Cost Calculation:**

```
A: Full External Sort:
  Sort Student (BR=100, B=22): 2 passes → 300 transfers → 50ms
  Read top 5 from sorted result: ~0.5ms
  Total: 50ms (wasteful — sorted entire file for 5 rows)

B: Priority Queue (in-memory heap of 5):
  Scan all 100 blocks: 100×0.1+4 = 14ms
  Maintain heap in memory → free
  Output top 5: free
  Total: 14ms

C: B+tree Index on gpa (HT=2), scan 5 from top:
  Traverse index to rightmost leaf: HT×(tT+tS) = 2×4.1 = 8.2ms
  Scan 5 leaf entries: 5×tT = 0.5ms (likely same page)
  Fetch 5 data records: 5×(tT+tS) = 5×4.1 = 20.5ms
  Total: 8.2+0.5+20.5 = 29.2ms
```

**Cost Comparison Table:**

| Algorithm | Cost (ms) | Notes |
|-----------|-----------|-------|
| Full sort | 50 | Wasteful for LIMIT k |
| Priority queue scan ★ | **14 ★** | One pass, O(n log k) |
| B+tree index scan | 29.2 | Good, but each fetch is random I/O |

> **Key Insight:** For **Top-K queries** (ORDER BY + LIMIT), a linear scan with an in-memory priority queue of size k achieves optimal I/O: one pass over the data, zero materializations. This beats both full sort and index scans when k is small and the index fetch cost is non-trivial.

---

### Example 57: Distinct Count — COUNT(DISTINCT)

**SQL Query:**
```sql
SELECT COUNT(DISTINCT cid) FROM Takes;
```

**Initial RA:**
```
γ_{COUNT(DISTINCT cid)}(Takes)
```

**Optimized RA:**
```
Option A: Sort Takes on cid, count distinct adjacent values
Option B: Hash Takes on cid, count distinct buckets
Option C: Use B+tree index on cid — count distinct keys from index leaf pages alone!
```

**Optimization Steps:**

- **Step 1:** Need COUNT of distinct cid values. V(cid, Takes) = 100 known from catalog → answer is 100! (catalog lookup).
- **Step 2:** In practice, V() is an estimate. For exact count, must process data.
- **Step 3:** Index-only scan: B+tree index on cid has cid values in leaf pages. Don't need to fetch actual Takes records — just scan index leaves.
- **Step 4:** This is called an **index-only scan** or **covering index**.

**Cost Calculation:**

```
If catalog is accurate: O(1) → answer = V(cid, Takes) = 100 instantly!
  Cost: 0ms (metadata lookup)

A: Sort + count distinct:
  Sort Takes (BR=200): 2 passes → 600 transfers ≈ 80ms
  Scan sorted, count distinct cid: 200×0.1+4 = 24ms
  Total: 104ms

B: Hash + count distinct:
  Scan Takes, hash on cid, count distinct entries:
  200×0.1+4 = 24ms  (small hash table, 100 entries)

C: Index-only scan on B+tree leaf pages for cid:
  Number of leaf pages in index ≈ BR_Takes / FR_index ≈ 200/25 = 8 leaf pages
  Cost: HT×(tT+tS) + leaf_pages×tT = 3×4.1 + 8×0.1 = 12.3+0.8 = 13.1ms
  No need to access base Takes records at all!
```

**Cost Comparison Table:**

| Method | Cost (ms) | Access pattern |
|--------|-----------|----------------|
| Catalog lookup | **0 ★★** | Metadata only |
| Index-only scan ★ | **13.1 ★** | Index leaves only |
| Hash scan | 24 | Full table scan |
| Sort scan | 104 | Full sort + scan |

> **Key Insight:** **Index-only scans** avoid touching the base table entirely — the index leaf pages contain all needed information. For aggregations (COUNT, MIN, MAX, DISTINCT) on indexed attributes, always check if the index alone can answer the query.

---

### Example 58: Star Schema Join (Data Warehouse Query)

**SQL Query:**
```sql
-- Fact table: Supply(sup_id, pid, qty, price)  NR=10000, BR=400
-- Dimensions already used in schema
SELECT SUM(SP.qty), S.city, P.color
FROM Supply SP, Supplier S, Parts P
WHERE SP.sup_id=S.sup_id AND SP.pid=P.pid
  AND S.city='Chicago' AND P.color='Red';
```

**Initial RA:**
```
γ_{S.city, P.color, SUM(qty)}(
  σ_{city='Chicago'}(Supplier) ⋈ Supply ⋈ σ_{color='Red'}(Parts)
)
```

**Optimized RA:**
```
γ_{SUM(qty)}(
  (σ_{city='Chicago'}(Supplier) ⋈_{sup_id} Supply)
  ⋈_{pid}
  σ_{color='Red'}(Parts)
)
-- Filter dimensions first, then join to fact table
```

**Optimization Steps:**

- **Step 1:** Push σ down to dimension tables first.
  - σ_{city='Chicago'}(Supplier): SC=ceil(500/20)=25 tuples → ~3 blocks.
  - σ_{color='Red'}(Parts): SC=ceil(800/8)=100 tuples → ~6 blocks.
- **Step 2:** Star join strategy: filter all dimensions first, then probe fact table.
- **Step 3:** Join order: small filtered dimensions first.
- **Step 4:** Filtered Supply rows: RF(city)=1/20=0.05, RF(color)=1/8=0.125 → Supply fraction ≈ 0.05×0.125=0.00625 → 63 Supply rows.

**Cost Calculation:**

```
σ_{city='Chicago'}(Supplier): 42×0.1+4 = 8.2ms → 25 tuples, ~3 blk
σ_{color='Red'}(Parts):        48×0.1+4 = 8.8ms → 100 tuples, ~6 blk

Hash join Supplier_Chicago(3) ⋈_{sup_id} Supply(400):
  3×(3+400) = 1209 → 1209×0.1+10×4 = 160.9ms
  Intermediate: Supply tuples with city='Chicago' ≈ 0.05×10000=500 tuples, 20 blk

Hash join intermediate(20) ⋈_{pid} Parts_Red(6):
  3×(20+6) = 78 → 78×0.1+10×4 = 47.8ms
  Intermediate: ≈63 rows

γ_{SUM(qty)} on 63 rows: free
Total: 8.2+8.8+160.9+47.8 = 225.7ms
```

**Cost Comparison Table:**

| Step | Operation | Cost (ms) | Output |
|------|-----------|-----------|--------|
| 1 | σ Supplier | 8.2 | 25 tuples |
| 2 | σ Parts | 8.8 | 100 tuples |
| 3 | Supplier ⋈ Supply | 160.9 | 500 tuples |
| 4 | Intermediate ⋈ Parts | 47.8 | 63 tuples |
| 5 | SUM aggregate | ~0 | 1 row |
| **Total** | | **225.7 ★** | |

> **Key Insight:** In star schema queries, **filter all dimension tables first** before joining to the large fact table. Each dimension filter dramatically reduces how much of the fact table is accessed. This "filter dimensions → join fact" pattern is the core of star join optimization.

---

### Example 59: View with Underlying Joins

**SQL Query (via view):**
```sql
CREATE VIEW CS_Students AS
  SELECT S.sid, S.sname, T.cid
  FROM Student S, Takes T
  WHERE S.sid=T.sid AND T.cid LIKE 'CS%';

SELECT sname FROM CS_Students WHERE sid = 42;
```

**Initial RA:**
```
σ_{sid=42}(π_{sid,sname,cid}(Student ⋈_{sid} σ_{cid LIKE 'CS%'}(Takes)))
```

**Optimized RA:**
```
σ_{sid=42} pushed through view expansion:
π_{sname}(σ_{sid=42}(Student) ⋈_{sid} σ_{cid LIKE 'CS%'}(Takes))
```

**Optimization Steps:**

- **Step 1:** View is expanded inline before optimization. No view materialization.
- **Step 2:** σ_{sid=42} can be pushed into the view definition: apply to Student (it's on sid which is Student's key).
- **Step 3:** σ_{sid=42}(Student): primary index on sid (HT=2) → 1 tuple.
- **Step 4:** Now join 1-tuple Student result with σ_{cid LIKE 'CS%'}(Takes).
- **Step 5:** LIKE 'CS%' covers cid values starting with CS. No index on cid (no prefix-matchable B+tree). Scan Takes, filter in memory.

**Cost Calculation:**

```
σ_{sid=42}(Student): A2 primary index → (2+1)×4.1 = 12.3ms → 1 tuple

σ_{cid LIKE 'CS%'}(Takes): scan Takes (200 blk)
  Cost: 200×0.1+4 = 24ms
  RF ≈ V(dept='CS')/V(cid): assume 30% are CS courses → 1500 tuples, 60 blk

INLJ: Student(1 tuple) outer, probe Takes result:
  With σ already applied to Takes → scan 1500 tuples
  Better: hash join 1 blk ⋈ 60 blk: 3×(1+60) = 183 → 22.3ms

π_{sname} on-the-fly: free

Total: 12.3 + 24 + 22.3 = 58.6ms
```

**Cost Comparison Table:**

| Plan | Cost (ms) | Notes |
|------|-----------|-------|
| Materialize view, then query | >200 | Never materialize for ad-hoc |
| Push σ through view ★ | **58.6 ★** | Inline view expansion + pushdown |

> **Key Insight:** Views should **always be expanded inline** during optimization — never materialize a view for an ad-hoc query. Selection pushdown through view boundaries is just as powerful as through base table operations.

---

### Example 60: Index-Only Scan for Join (Covering Index)

**SQL Query:**
```sql
SELECT T.cid, COUNT(*) FROM Takes T
WHERE T.sid IN (SELECT sid FROM Student WHERE gpa > 3.8)
GROUP BY T.cid;
```

**Initial RA:**
```
γ_{cid, COUNT(*)}(Takes ⋈_{sid} σ_{gpa>3.8}(Student))
```

**Optimized RA:**
```
If composite index on Takes(sid, cid) exists:
  Index-only scan for the semi-join
  No need to fetch Takes base records!
```

**Optimization Steps:**

- **Step 1:** Flatten IN subquery to semi-join.
- **Step 2:** σ_{gpa>3.8}(Student): secondary index on gpa. SC ≈ 5% × 1000 = 50 tuples.
- **Step 3:** Build hash set of 50 qualifying sids.
- **Step 4:** Need (sid, cid) from Takes for the GROUP BY. If composite index on (sid, cid) exists: scan only index leaf pages (no base table fetch needed).
- **Step 5:** Index-only scan: index contains (sid, cid) pairs — sufficient to check sid ∈ hash set and return cid.

**Cost Calculation:**

```
Step 1: σ_{gpa>3.8}(Student):
  A6 secondary index: estimate 50 tuples below 3.8 in top 5%?
  Actually gpa>3.8: A4 secondary index (HT=2, SC≈50)
  Cost = (2+50)×4.1 = 213.2ms? → linear scan better: 14ms
  Build hash set of 50 sids: free

Step 2: Index-only scan on Takes(sid,cid), HT=3:
  Leaf pages ≈ NR_Takes / (block_size / entry_size)
              = 5000 / (512/8) = 5000/64 ≈ 78 leaf pages
  Traverse to first leaf: HT×(tT+tS) = 3×4.1 = 12.3ms
  Scan all 78 leaf pages: 78×tT = 7.8ms (sequential after first seek)
  For each leaf entry: check sid ∈ hash set, if yes emit cid
  Total index-only: 12.3+7.8 = 20.1ms

Step 3: γ_{cid, COUNT(*)} on emitted cids (≤50×5=250 entries): free (in-memory)

TOTAL: 14 + 20.1 + 0 = 34.1ms

Without index-only scan (scan base Takes):
  200×0.1+4 = 24ms (cheaper here since base table not much bigger than index!)
  + 14ms (σ Student) = 38ms
```

**Cost Comparison Table:**

| Plan | Cost (ms) | |
|------|-----------|--|
| Scan Student + Scan Takes | 38 | No index exploitation |
| Scan Student + Index-only Takes ★ | **34.1 ★** | Marginal improvement here |

> **Key Insight:** **Index-only scans** shine for large tables where the index is much smaller than the base table, or when the base table I/O would be random (secondary index). Here the benefit is modest because Takes base table is similar in size to the index leaves.

---

### Example 61: Materialized View Usage

**SQL Query (frequent):**
```sql
-- Run 100 times per day:
SELECT dept, COUNT(*) as student_count, AVG(gpa) as avg_gpa
FROM Student S, Takes T, Course C
WHERE S.sid=T.sid AND T.cid=C.cid
GROUP BY dept;
```

**Optimized Strategy:**
```sql
-- Create materialized view:
CREATE MATERIALIZED VIEW dept_stats AS
  SELECT dept, COUNT(*) as student_count, AVG(gpa) as avg_gpa
  FROM Student S, Takes T, Course C
  WHERE S.sid=T.sid AND T.cid=C.cid
  GROUP BY dept;

-- Query becomes: SELECT * FROM dept_stats;
```

**Optimization Steps:**

- **Step 1:** Query is complex (3 joins + aggregate) but runs frequently.
- **Step 2:** Result is small (V(dept)=5 rows). Can be materialized.
- **Step 3:** Materialized view: precomputed result stored on disk. Query cost = 1 block read.
- **Step 4:** Maintenance cost: re-compute when Student/Takes/Course change.
- **Step 5:** Break-even: if query runs ≥ k times before data changes, materialized view wins.

**Cost Calculation:**

```
Ad-hoc query cost (each execution):
  Step 1: σ pushed, join Course⋈Takes: 3×(9+200)=627 → 90ms
  Step 2: Intermediate(200 blk) ⋈ Student(100): 3×300=900 → 130ms
  Step 3: GROUP BY dept: 24ms (hash aggregation on 5 groups)
  Total per execution: 244ms

Materialized view:
  Build cost (once): 244ms
  Query cost: read 1 block ≈ 0.1+4 = 4.1ms

Break-even: 244 × k > 244 + k × 4.1
  k > 1.02 → any 2+ executions → materialized view wins!

For 100 executions/day:
  Without MV: 100 × 244 = 24,400 ms/day
  With MV:    244 + 100 × 4.1 = 244 + 410 = 654 ms/day
  Speedup: 37×
```

**Cost Comparison Table:**

| Strategy | Per query (ms) | 100×/day (ms) | |
|----------|---------------|--------------|--|
| Ad-hoc re-computation | 244 | 24,400 | |
| Materialized view ★ | **4.1 ★** | **654 ★** | 37× speedup |

> **Key Insight:** Materialized views amortize expensive join+aggregate costs over many queries. Break-even occurs at just **2 queries** — making them almost always worthwhile for frequently-run analytical queries, provided the data doesn't change every query.

---

### Example 62: Partition Pruning (Range Partitioning)

**SQL Query:**
```sql
-- Employee is range-partitioned on salary into 5 partitions:
-- P1: <30k, P2: 30k-50k, P3: 50k-80k, P4: 80k-120k, P5: >120k
SELECT * FROM Employee WHERE salary BETWEEN 50000 AND 80000;
```

**Initial RA:**
```
σ_{salary≥50000 ∧ salary≤80000}(Employee)
```

**Optimized RA:**
```
σ_{salary≥50000 ∧ salary≤80000}(Employee_P3)  -- only scan partition P3!
-- Partition pruning: only P3 (50k-80k) contains qualifying tuples
```

**Optimization Steps:**

- **Step 1:** Employee is range-partitioned on salary. Each partition holds 1/5 of rows.
- **Step 2:** Partition P3 covers exactly 50k–80k → our range falls entirely within P3.
- **Step 3:** **Partition pruning**: only scan P3. Partitions P1, P2, P4, P5 are provably empty for this query.
- **Step 4:** P3 size: BR/5 = 200/5 = 40 blocks.

**Cost Calculation:**

```
Without partition pruning (scan all Employee):
  Cost = 200×0.1+4 = 24.0ms

With partition pruning (scan only P3):
  P3 size: BR/5 = 40 blocks
  Cost = 40×0.1+4 = 4+4 = 8.0ms  ← 3× better

Secondary index (no partitioning):
  n ≈ 2000×0.3=600 tuples → (2+600)×4.1 = 2468ms ← terrible

Index + Partition Pruning (best):
  Only 40 blocks in P3, with clustering on salary:
  SC in P3 = 400 tuples (all qualify → essentially scan P3)
  Cost ≈ 8.0ms (scan P3 directly)
```

**Cost Comparison Table:**

| Strategy | Blocks scanned | Cost (ms) | |
|----------|---------------|-----------|--|
| Full table scan | 200 | 24.0 | |
| Partition pruning ★ | 40 | **8.0 ★** | 3× speedup |
| Secondary index | 600 random I/Os | 2468 | ❌ |

> **Key Insight:** **Partition pruning** is a physical optimization where the query optimizer eliminates entire partitions that cannot contain qualifying tuples. For range queries on the partition key, this provides a speedup equal to the number of partitions: here 5×.

---

### Example 63: Parallel Query — Partition Parallelism

**SQL Query:**
```sql
-- Employee hash-partitioned on dno across 5 nodes
-- Each node has: NR=400, BR=40
SELECT dno, SUM(salary) FROM Employee GROUP BY dno;
```

**Initial RA:**
```
γ_{dno, SUM(salary)}(Employee)   -- distributed across 5 nodes
```

**Optimized RA:**
```
-- Local aggregation on each node (partial sums per dno),
-- then global merge at coordinator node
```

**Optimization Steps:**

- **Step 1:** Employee hash-partitioned on dno → each node has all records for a subset of dno values.
- **Step 2:** Since partitioning is on dno (the GROUP BY attribute), each node computes COMPLETE aggregates for its dno values.
- **Step 3:** No inter-node shuffle needed! Each node computes final SUM(salary) for its dno groups.
- **Step 4:** Coordinator collects results (5 rows per node × 10 groups / 5 nodes = 2 rows/node avg).
- **Step 5:** Total wall-clock time ≈ time for one node (parallel execution).

**Cost Calculation:**

```
Sequential (one node, full table):
  Scan 200 blocks + hash aggregate: 24ms

Parallel (5 nodes, partitioned on dno):
  Each node scans its 40 blocks: 40×0.1+4 = 8ms
  Local hash aggregate (in-memory): free
  Network: 5 nodes × 2 avg rows × tiny row size ≈ 0ms
  Coordinator merge: 10 rows ← instant
  Wall-clock time: 8ms  (3× speedup, 5 nodes)

Parallel (5 nodes, partitioned on eid NOT dno):
  Each node has all dno values → must shuffle:
    Each node scans 40 blocks: 8ms
    Network shuffle by dno: redistribute rows → each node sends 400/5=80% elsewhere
    Network cost: non-trivial → 5-10ms
    Re-aggregate after shuffle: small → 2ms
    Wall-clock: ~20ms (less efficient due to shuffle)
```

**Cost Comparison Table:**

| Strategy | Wall-clock (ms) | Network | |
|----------|----------------|---------|--|
| Sequential | 24 | None | |
| Parallel (right partition key) ★ | **8 ★** | Zero shuffle | |
| Parallel (wrong partition key) | ~20 | Full shuffle | |

> **Key Insight:** When data is **partitioned on the GROUP BY attribute**, aggregation is embarrassingly parallel — each node computes final results independently, requiring no network shuffle. Choosing the right partition key for your most common query patterns is critical in distributed databases.

---

### Example 64: Bushy Tree vs Left-Deep Tree — When Bushy Wins

**SQL Query:**
```sql
SELECT * FROM Student S, Takes T, Course C, Department D
WHERE S.sid=T.sid AND T.cid=C.cid AND C.dept=D.dname
  AND D.budget > 500000;
```

**Initial RA:**
```
(Student ⋈ Takes ⋈ Course ⋈ Department) ∧ σ_{budget>500k}(Department)
```

**Left-Deep Tree:**
```
((Student ⋈ Takes) ⋈ Course) ⋈ σ_Dept
```

**Bushy Tree:**
```
(Student ⋈ Takes) ⋈ (σ_Dept ⋈ Course)
-- Dept and Course can be joined in parallel with Student and Takes!
```

**Optimization Steps:**

- **Step 1:** Dept after σ_{budget>500k}: 50×0.5=25 tuples ≈ 2 blocks.
- **Step 2:** σ_Dept ⋈ Course: 2 blocks ⋈ 9 blocks = small (25×20=500 tuples ≈ 42 blocks).
- **Step 3:** Left-deep: (S⋈T)⋈C first → large intermediate → then join with small σ_Dept.
- **Step 4:** Bushy: small Dept⋈Course in parallel with large S⋈T, then join two moderate-sized intermediates.
- **Step 5:** Bushy allows PARALLEL execution of the two sub-joins!

**Cost Calculation:**

```
Left-Deep Tree: ((S⋈T)⋈C)⋈D_filtered
  S⋈T: 3×(100+200)=900 → 130ms; result=5000 tuples=200 blk
  (200)⋈C(9): 3×209=627 → 90ms; result=5000 tuples=200 blk
  (200)⋈D_f(2): 3×202=606 → 90ms
  Total sequential: 130+90+90 = 310ms

Bushy Tree: (S⋈T) ⋈ (D_filtered⋈C)
  Branch 1 (S⋈T): 3×300=900 → 130ms; result=5000 tuples=200 blk
  Branch 2 (D_f⋈C): 3×(2+9)=33 → ~17ms; result=500 tuples=42 blk
  [Branches execute in parallel]
  Final join (200⋈42): 3×242=726 → 104ms
  
  Sequential cost: 130+17+104 = 251ms
  Parallel (Branch 1 || Branch 2): max(130,17)+104 = 130+104 = 234ms

Left-deep sequential: 310ms
Bushy sequential: 251ms  (18% faster)
Bushy parallel: 234ms   (24% faster)
```

**Cost Comparison Table:**

| Tree Type | Sequential (ms) | Parallel (ms) | |
|-----------|----------------|--------------|--|
| Left-Deep | 310 | 310 | No natural parallelism |
| Bushy ★ | 251 | **234 ★** | Parallel sub-joins |

> **Key Insight:** Bushy trees allow **parallel execution** of independent sub-joins, which left-deep trees cannot. System R restricts to left-deep trees to limit plan search space, but modern parallel DBMS (e.g., Spark SQL, Greenplum) consider bushy plans for their parallelism benefits.

---

### Example 65: Complete 4-Table Query — All Optimizations Applied

**SQL Query:**
```sql
SELECT S.sname, C.cname, T.grade, D.dname
FROM Student S, Takes T, Course C, Department D
WHERE S.sid=T.sid AND T.cid=C.cid AND C.dept=D.dname
  AND S.gpa > 3.5 AND C.credits = 4 AND D.budget > 200000
ORDER BY T.grade DESC;
```

**Step-by-Step Optimization:**

**Step 1 — Parse and Translate (Canonical RA):**
```
π_{sname,cname,grade,dname}(
  σ_{gpa>3.5 ∧ credits=4 ∧ budget>200k}(
    Student × Takes × Course × Department
  )
)
```

**Step 2 — Break Conjunctive σ (Rule 1):**
```
σ_{gpa>3.5} ∧ σ_{credits=4} ∧ σ_{budget>200k}
```

**Step 3 — Push σ Down (Rule 6):**
```
σ_{gpa>3.5}(Student)    → est. 250 tuples, 25 blk
σ_{credits=4}(Course)   → SC(credits)=25, est. 25 tuples, 3 blk
σ_{budget>200k}(Dept)   → est. 50×0.6=30 tuples, 2 blk
```

**Step 4 — Estimate Sizes and Order Joins:**
```
Smallest after filter: Dept(2 blk) < Course_cr4(3 blk) < Student_gpa(25 blk) < Takes(200 blk)
Join order: ((Dept ⋈ Course_cr4) ⋈ Takes) ⋈ Student_gpa
```

**Step 5 — Push π Down (Rule 7):**
```
π_{dname,dept} from Dept+Course (only needed: dname, cid, cname)
π_{sid,cid,grade} from Takes
π_{sid,sname} from Student_gpa
```

**Step 6 — Identify Pipelineable Subtrees:**
```
σ and π applied on-the-fly; hash joins materialize partitions only
ORDER BY sort is last — sort final result (small) on grade
```

**Cost Calculation:**

```
σ_{budget>200k}(Dept):   3×0.1+4 = 4.3ms  → 30 tuples, 2 blk
σ_{credits=4}(Course):   9×0.1+4 = 4.9ms  → 25 tuples, 3 blk
σ_{gpa>3.5}(Student):   100×0.1+4 = 14ms  → 250 tuples, 25 blk

Join 1: σ_Dept(2) ⋈_{dept=dname} σ_Course(3):
  Hash join: 3×(2+3)=15 → 15×0.1+10×4=41.5ms
  Result: 30×25/V(dept)=30×25/5=150 tuples ≈ 13 blk (pushed π: dept+cid+cname+dname)

Join 2: DeptCourse(13) ⋈_{cid} Takes(200):
  Hash join: 3×(13+200)=639 → 639×0.1+10×4=103.9ms
  Result: 150×50=7500? No — cid is key for Course → ≤ Takes rows
  Better: NR_Takes × (qualifying Course/total Course) = 5000×25/100=1250 tuples ≈ 50 blk

Join 3: DeptCourseTakes(50) ⋈_{sid} Student_gpa(25):
  Hash join: 3×(50+25)=225 → 225×0.1+10×4=62.5ms
  Result: min(1250,250)≈250 tuples (gpa filter is selective)

π on-the-fly: free
ORDER BY grade (250 tuples ≈ 25 blk, fits in memory): 25×0.1+1×4 = 6.5ms

TOTAL: 4.3+4.9+14+41.5+103.9+62.5+6.5 = 237.6ms
```

**Cost Comparison Table:**

| Stage | Operation | Cost (ms) | Output |
|-------|-----------|-----------|--------|
| 1 | σ Dept | 4.3 | 30 tuples |
| 2 | σ Course | 4.9 | 25 tuples |
| 3 | σ Student | 14.0 | 250 tuples |
| 4 | Dept ⋈ Course | 41.5 | 150 tuples |
| 5 | DeptCourse ⋈ Takes | 103.9 | 1250 tuples |
| 6 | + Student | 62.5 | 250 tuples |
| 7 | ORDER BY | 6.5 | 250 tuples |
| **TOTAL** | | **237.6 ★** | **250 rows** |

> **Key Insight:** All 6 heuristic steps together reduce a 4-table Cartesian product (billions of rows) to a 238ms query. **Pushing σ first** is the most impactful step (each σ reduced a relation by 50–75%). **Pushing π** reduces intermediate record sizes (not shown separately but saves ~15% more I/O). The ORDER BY sort is cheap because the final result is small.

---

## SECTION D — Advanced Topics (Examples 66–80)

---

### Example 66: Statistics Update Impact

**Scenario:** Catalog statistics are **stale** — the optimizer makes wrong decisions.

```sql
-- Catalog says: NR(Student)=1000, V(gpa)=20, SC(gpa=4.0)=50
-- Reality: 80% of students have gpa=4.0 after grade inflation!
-- Actual SC(gpa=4.0) = 800

SELECT * FROM Student WHERE gpa = 4.0;
```

**Impact of Stale Statistics:**

```
Optimizer believes:
  SC(gpa=4.0) = 50 → uses secondary index (HT=2)
  Estimated cost: (2+50)×4.1 = 213.2ms

Actual execution:
  SC(gpa=4.0) = 800 → needs to fetch 800 records (possibly 800 blocks!)
  Actual cost: (2+800)×4.1 = 3,290.2ms

Linear scan (correct choice if stats were fresh):
  Cost: 14ms
  With accurate stats: optimizer would choose A1

Stale statistics caused a 3290/14 = 235× slowdown!
```

**Cost Comparison:**

| Statistics | Chosen Plan | Actual Cost (ms) | Optimal (ms) |
|-----------|------------|-----------------|-------------|
| Stale (SC=50) | A4 Index | 3,290 | — |
| Fresh (SC=800) | A1 Linear ★ | **14 ★** | 14 |

> **Key Insight:** Stale statistics can cause catastrophically wrong plan choices. Many DBMS provide `ANALYZE TABLE` or `UPDATE STATISTICS` commands. Modern systems like PostgreSQL auto-analyze frequently updated tables, and some use **adaptive query processing** to correct plans mid-execution.

---

### Example 67: Nested Loop Join with Buffer Pool Analysis

**SQL Query:**
```sql
SELECT * FROM Parts P, Supply SP WHERE P.pid = SP.pid;
-- Parts: BR=48, NR=800. Supply: BR=400, NR=10000. B=10 buffer pages.
```

**Buffer Allocation Analysis:**

```
BNLJ with B=10 buffer pages:
  Strategy: allocate B-2=8 pages for outer chunks, 1 for inner, 1 for output
  Outer: Parts (smaller, BR=48)
  Chunks: ceil(48/8) = 6 chunks
  Supply scanned once per chunk: 6 × 400 + 48 = 2448 transfers
  Cost = 2448×0.1 + (6+6)×4 = 244.8 + 48 = 292.8ms

Alternative allocation: allocate more to output buffer
  Say: 6 outer, 1 inner, 3 output
  Output buffer only affects write cost (final output), not join cost
  Join cost unchanged: 292.8ms

Alternative: allocate to inner buffer (for sequential inner reads)?
  Inner is already read sequentially per chunk → extra inner buffers help only
  if we can prefetch → minor improvement, negligible

Optimal with B=10: Parts outer, 8-page chunks = 292.8ms

Hash Join with B=10:
  Phase 1: 9 partitions. Parts partitions ≈ 48/9 = 6 blk each.
  Check: can Parts partition (6 blk) fit in B-2=8 pages? YES!
  Cost: 3×(48+400) = 1344 transfers
  = 1344×0.1 + seeks ≈ 134.4 + 40 = 174.4ms  ← better
```

**Cost Comparison:**

| Algorithm | B=10 Cost (ms) | B=5 Cost (ms) |
|-----------|---------------|--------------|
| BNLJ (Parts outer) | 292.8 | ~580 |
| Hash Join ★ | **174.4 ★** | ~350 (recursive) |

> **Key Insight:** Buffer allocation matters significantly for BNLJ: allocate **B-2 pages to the outer chunk** to maximize chunk size and minimize re-scans of the inner relation. Hash join is generally less sensitive to exact buffer allocation as long as partitions fit in memory.

---

### Example 68: Multi-Attribute Sort — Composite Sort Key

**SQL Query:**
```sql
SELECT * FROM Employee ORDER BY dno ASC, salary DESC;
```

**Initial RA:**
```
SORT_{dno ASC, salary DESC}(Employee)
```

**Optimization Steps:**

- **Step 1:** Composite sort key (dno ASC, salary DESC). Sort on dno first, then within each dno group, sort by salary descending.
- **Step 2:** A B+tree index on (dno, salary) could support this if both are ASC/DESC compatible.
- **Step 3:** External sort with composite key: same algorithm, comparator handles both attributes.

**Cost Calculation:**

```
External Sort (BR=200, B=22):
  Pass 0: ceil(200/22)=10 runs; Merge: ceil(log21(10))=1 pass; Total=2 passes
  Cost: 2×200×2-200 = 600 transfers
  Each comparison uses (dno ASC, salary DESC) comparator — same I/O, slightly slower CPU
  Cost ≈ 600×0.1 + seeks ≈ 60+20 = 80ms

Index-based sort (B+tree on (dno, salary ASC)):
  Problem: index on salary ASC, but we need DESC → scan index backward
  B+trees support backward scans! → cost same as forward scan
  Index traversal + leaf page scan: HT×(tT+tS) + leaf_pages×tT
  ≈ 2×4.1 + 20×0.1 = 8.2+2 = 10.2ms (traverse leaves, get sorted record pointers)
  + fetch 2000 records: 2000 record pointers, each possibly different block → random I/O!
  2000×4.1 = 8200ms → MUCH worse than external sort

Index feasible only if Employee is CLUSTERED on (dno, salary):
  Then sequential read of clustering index = BR×tT + tS = 24ms (but already sorted!)
```

**Cost Comparison:**

| Method | Cost (ms) | Notes |
|--------|-----------|-------|
| External sort ★ | **80 ★** | For non-clustered data |
| Index-only (non-clustered) | 8200 | 2000 random I/Os — terrible |
| Clustered index ★★ | **24 ★★** | Already sorted — free! |

> **Key Insight:** For a composite sort key (dno, salary), the external sort cost is identical to a single-attribute sort — only the comparator changes. An index only helps if the table is physically sorted (clustered) on the composite key.

---

### Example 69: Cost of Writing Final Output

**SQL Query:**
```sql
SELECT * FROM Employee WHERE dno=3;
-- Result written to disk (large client transfer assumed)
```

**Discussion: Why Final Output Cost is Usually Excluded**

```
Standard formula: Cost = b×tT + S×tS   [for INPUT/intermediate I/O only]
Writing final output: typically excluded from optimizer's cost model because:
  1. Cost depends on network speed (client location unknown)
  2. Client might process results on-the-fly (streaming)
  3. Destination might be memory buffer (no disk write needed)

If we DID include output write cost:
  σ_{dno=3}(Employee): 200 qualifying tuples = 20 output blocks
  Write cost: 20×tT + tS = 20×0.1+4 = 6ms  [additional]

  A1 scan: 24ms (input) + 6ms (output) = 30ms
  A3 clustering: 14.2ms (input) + 6ms (output) = 20.2ms
  
  Relative ranking: unchanged — output cost is constant for both

Output cost matters for: INSERT INTO ... SELECT, CREATE TABLE AS SELECT (CTAS)
  Then: output write IS included in planning cost
```

**Cost With and Without Output:**

| Plan | Input Only (ms) | + Output (ms) | Notes |
|------|----------------|--------------|-------|
| A1 Scan | 24.0 | 30.0 | Always add output for CTAS |
| A3 Clustering ★ | 14.2 ★ | **20.2 ★** | Relative order unchanged |

> **Key Insight:** Output write cost is **excluded from standard cost formulas** because it's typically constant regardless of which input plan is chosen. However, for CTAS (CREATE TABLE AS SELECT) or INSERT INTO ... SELECT statements, output write cost should be included.

---

### Example 70: Adaptive Query Processing

**Scenario:** Mid-query plan correction when estimates are wrong.

```sql
SELECT * FROM Student S, Takes T WHERE S.sid=T.sid AND S.gpa>3.5;
```

**Initial Plan (based on estimates):**
```
Optimizer estimates: SC(gpa>3.5, Student) = 250 tuples → chose hash join
  Estimated hash join cost: 3×(25+200) = 675 transfers ≈ 130ms
```

**Runtime Discovery:**
```
Actual: σ_{gpa>3.5}(Student) produces 800 tuples (not 250)
  Optimizer's RF estimate of 25% was wrong — actual RF = 80%!
```

**Adaptive Response:**
```
Non-adaptive (continue with original plan):
  Hash join with 800-tuple Student side (80 blk):
  3×(80+200) = 840 transfers ≈ 130ms (coincidentally same; here 80 blk ~ 25 blk approx)
  Actually worse if partitions don't fit: overflow issues

Adaptive plan switch:
  At runtime, detect that Student side is 80 blk not 25 blk
  Switch to sort-merge join (less sensitive to input size changes)
  Sort Student (80 blk, B=22): 2 passes → 240 transfers ≈ 40ms
  Sort Takes (200 blk): 600 transfers ≈ 80ms
  Merge: 280 transfers ≈ 36ms
  Total: 40+80+36 = 156ms

PostgreSQL-style row estimates: 
  if actual_rows > 2× estimated_rows → plan may be suboptimal
  but replanning mid-query is expensive (not commonly done in traditional RDBMS)
  Eddies, Rio, and other adaptive QP systems do support mid-query plan changes
```

**Cost Comparison:**

| Strategy | Estimated cost | Actual cost | Error |
|----------|---------------|-------------|-------|
| Original hash join | 130ms | 170ms | 31% |
| Adaptive sort-merge ★ | — | **156ms ★** | Better |

> **Key Insight:** Adaptive query processing (AQP) corrects for wrong selectivity estimates by **switching plans mid-execution** based on actual cardinalities observed at runtime. While most RDBMS don't support true mid-query plan switches, PostgreSQL 14+ introduced better adaptive statistics collection.

---

## SECTION E — Final 10 Comprehensive Examples (71–80)

---

### Example 71: Covering Index for JOIN

```sql
SELECT T.cid, T.grade FROM Takes T WHERE T.sid = 500;
-- Assume composite index on Takes(sid, cid, grade) — all needed attrs covered!
```

```
Without covering index:
  Secondary index on sid (HT=3): locate 5 records, fetch from base table
  Cost = (3+5)×4.1 = 32.8ms

With covering index (sid, cid, grade) — index-only scan:
  All needed attributes (sid, cid, grade) are IN the index leaf entries
  No base table access needed!
  Cost = HT×(tT+tS) + ceil(SC/entries_per_leaf_page)×tT
       = 3×4.1 + 1×0.1 = 12.3+0.1 = 12.4ms
  Savings: 32.8 - 12.4 = 20.4ms (62% faster)
```

> **Key Insight:** A **covering index** (also called index-only scan) stores all columns needed by the query in the index itself. No base table fetch is required, eliminating the most expensive part of secondary index access (random I/Os per record).

---

### Example 72: Function-Based Index Limitation

```sql
SELECT * FROM Student WHERE UPPER(sname) = 'ALICE';
-- Regular index on sname CANNOT be used (UPPER() transforms the key)
```

```
Regular B+tree on sname:
  Cannot use! Index is on raw sname values ('Alice', 'alice', 'ALICE' sorted differently)
  UPPER(sname) is not the index key → must scan all tuples

A1 Linear scan:
  Cost = 100×0.1+4 = 14ms
  Apply UPPER(sname)='ALICE' in memory

Function-based index on UPPER(sname) [if created]:
  Index stores UPPER(sname) as the key
  Cost = (HT+n)×(tT+tS) = (2+1)×4.1 = 12.3ms (assuming few students named ALICE)

Without function-based index: always A1 = 14ms
```

> **Key Insight:** Functions applied to indexed attributes in WHERE clauses **disable the index** unless a function-based index is explicitly created. This is a common performance pitfall: `WHERE LOWER(email) = 'x'` will NOT use an index on `email`.

---

### Example 73: OR-to-UNION Rewrite

```sql
-- Sometimes OR queries can be rewritten as UNION for index exploitation
SELECT * FROM Employee WHERE dno=3 OR eid=500;
-- A10 (union of identifiers) vs UNION rewrite
```

```
A10 — Union of identifiers:
  Index on dno (HT=2, SC=200): get 200 RIDs
  Index on eid (HT=2, SC=1 — key): get 1 RID
  Union: 201 unique RIDs (remove duplicate if eid=500 is in dno=3)
  Fetch 201 records: 201×4.1 = 824.1ms + index scan overhead
  Total ≈ 852ms

A1 — Linear scan:
  Cost = 24ms  ← wins decisively

UNION rewrite (hypothetical speedup scenario if both SC were tiny):
  (SELECT * FROM Employee WHERE dno=3)
  UNION
  (SELECT * FROM Employee WHERE eid=500)
  → Two separate plans, deduplicate
  Still: dno index fetch 200 records = 828ms → worse than scan
```

> **Key Insight:** OR conditions with low-selectivity predicates are better served by **linear scan** than by A10 or UNION rewrites. UNION rewrite only helps when BOTH branches are highly selective (SC ≤ 3 each).

---

### Example 74: Predicate Ordering in Linear Scan

```sql
SELECT * FROM Employee WHERE salary > 100000 AND dno = 3 AND age < 30;
-- All conditions: no indexes. What order to evaluate predicates?
```

```
Linear scan evaluates one block at a time. For each record, evaluate predicates.
If any predicate fails: skip record immediately (short-circuit evaluation).

Predicate Costs (CPU, negligible vs I/O):
  All predicates evaluated in memory → predicate order affects CPU, not I/O

Optimal predicate order (most selective first → reject early):
  RF(salary>100k) = 0.1  (only 10% qualify — very selective!)
  RF(dno=3)       = 0.1  (10% qualify)
  RF(age<30)      = 0.25 (25% qualify)

Order: salary>100k first (RF=0.1), then dno=3 (RF=0.1), then age<30
  Expected record evaluations before rejection:
    1/RF_first = 1/0.1 = 10 records examined per qualifying record
    With optimal order: 10 comparisons per record on average
  
  vs worst order (age first, RF=0.25):
    4 comparisons per record on average

I/O Cost (same for all orders):
  200×0.1+4 = 24ms  [ordering only affects CPU time]

Key formula for predicate ordering:
  Order predicates by (1-RF)/cost_to_evaluate ascending
  (evaluate cheap, highly selective predicates first)
```

> **Key Insight:** Predicate ordering within a linear scan is a **CPU optimization** (I/O cost is unchanged). Apply the most selective, cheapest-to-evaluate predicates first to short-circuit evaluation as early as possible.

---

### Example 75: Join Attribute Data Type Mismatch

```sql
SELECT * FROM Student S, Takes T WHERE S.sid = T.sid;
-- Disaster scenario: sid is INTEGER in Student but VARCHAR in Takes!
```

```
Type mismatch: INTEGER sid vs VARCHAR sid
  DBMS must cast one type to the other for comparison
  If cast applied to indexed column → index becomes UNUSABLE

Scenario A: Cast T.sid to INTEGER
  CAST(T.sid AS INTEGER) = S.sid
  Index on Takes.sid (VARCHAR) is now unusable!
  → Full scan of Takes required

Scenario B: Cast S.sid to VARCHAR  
  S.sid::VARCHAR = T.sid
  Index on Student.sid (INTEGER) is unusable!
  → Full scan of Student required

Cost Impact:
  BNLJ without any useful index: 100×200+100 = 20,100 transfers → 2414ms
  vs.
  Hash Join (works regardless of type — just cast during hash phase): 900 → 130ms
  
  Hash join handles type mismatch gracefully (cast during hashing).
  NLJ/BNLJ + index cannot exploit indexes with type mismatch.
```

> **Key Insight:** **Data type mismatches** in join conditions disable indexes on both sides. This is a schema design problem: always use the same data type for join attributes. Hash join is more robust to type mismatches since it performs the cast during the hash partitioning phase.

---

### Example 76: Multi-Column Join Condition

```sql
SELECT * FROM Supply SP1, Supply SP2
WHERE SP1.sup_id = SP2.sup_id AND SP1.pid = SP2.pid AND SP1.qty > SP2.qty;
-- Find pairs of supply records with same supplier+part but different quantities
```

```
This is a SELF-JOIN with multi-column join condition (sup_id, pid) plus a filter.

Composite index on Supply(sup_id, pid) matches the join condition!

INLJ approach:
  For each SP1 tuple, probe composite index for (SP1.sup_id, SP1.pid):
    SC = NR_Supply / (V(sup_id)×V(pid)) = 10000/(500×800) ≈ 0.025 → ~1 tuple
    But wait: it's a self-join, same table → same bucket ≈ 1-2 matching SP2 tuples
  Cost = BR_Supply + NR_Supply × c
       = 400×0.1 + 1×4 + 10000 × (HT+1)×(tT+tS)
       = 44 + 10000×4×4.1 = 44+164000 → terrible for composite index self-join!

Hash Join on (sup_id, pid):
  3×(400+400) = 2400 transfers (reading Supply twice: once as SP1, once as SP2)
  Cost ≈ 2400×0.1+20×4 = 240+80 = 320ms
  Filter qty>SP2.qty applied on-the-fly during probe
```

> **Key Insight:** Self-joins with multi-column conditions are handled efficiently by hash join on the composite key. INLJ is impractical when the table is large (10000 rows × index probe cost = huge).

---

### Example 77: Subquery with Aggregate — MAX/MIN Optimization

```sql
SELECT * FROM Employee WHERE salary = (SELECT MAX(salary) FROM Employee);
```

```
Block 2 (inner): SELECT MAX(salary) FROM Employee
  Option A: Linear scan + MAX aggregation:
    Scan 200 blocks, maintain running max: 200×0.1+4 = 24ms
  Option B: Secondary B+tree on salary (HT=2), scan to rightmost leaf entry:
    Traverse to max: HT×(tT+tS) + 1 leaf entry = 2×4.1+0.1 = 8.3ms  ← MUCH BETTER!
    Index-based MAX: follow rightmost path in B+tree, read rightmost leaf entry = 1 value
  
Block 1 (outer): σ_{salary=MAX_VALUE}(Employee)
  After substituting MAX result (say 150000):
  Secondary index on salary (HT=2), SC(salary=150k)=4 (maybe just 1 person at max)
  Cost = (2+1)×4.1 = 12.3ms (if unique max) or (2+n)×4.1

Total optimized: 8.3 + 12.3 = 20.6ms
Total naive (scan + scan): 24 + 24 = 48ms
```

> **Key Insight:** **B+tree indexes support O(log n) MIN/MAX retrieval**: just follow the leftmost (MIN) or rightmost (MAX) path from root to leaf. This is dramatically faster than a full scan for aggregate queries.

---

### Example 78: Cached Plan Reuse

```sql
-- Query template (prepared statement):
SELECT * FROM Employee WHERE dno = ?;  -- parameter: dno_value

-- First execution: dno=3 → SC=200 → optimizer chooses A1 (linear scan)
-- Second execution: dno=3 → same plan → correct (SC=200)  
-- Third execution: dno=99 (non-existent dno!) → SC=0 → plan still A1 → correct but wasteful?
```

```
Plan caching tradeoff:
  Benefit: avoid re-optimization cost (can be expensive for complex queries)
  Risk: optimal plan may differ per parameter value

Example:
  dno=3:   SC=200/2000=10% → A1 scan (24ms) better than secondary index (828ms)
  dno=7:   SC=200/2000=10% → same, A1 still correct
  
  If hypothetical dno with only 2 employees:
    SC=2 → secondary index: (2+2)×4.1=16.4ms vs scan: 24ms → index better!
    But cached plan uses A1 → 24ms (not terrible, just suboptimal)

Re-optimization triggers:
  PostgreSQL: re-plan after 5 executions if row estimates differ >2× actual
  Oracle: adaptive cursor sharing — different plans for different bind values
  SQL Server: parameter sniffing — first execution's plan is reused
```

> **Key Insight:** **Parameter sniffing** (caching plans based on first parameter value) can cause performance issues when the same query is run with very different selectivities. Solutions include plan hints, OPTION(RECOMPILE), or adaptive cursor sharing.

---

### Example 79: NULL Handling in Selection and Join

```sql
SELECT * FROM Employee WHERE salary IS NULL;
SELECT * FROM Student S, Takes T WHERE S.sid = T.sid;  -- NULL sids?
```

```
NULL selection:
  σ_{salary IS NULL}(Employee)
  No B+tree index stores NULL values (NULLs are excluded from standard indexes)!
  → Must use A1 (linear scan)
  Cost: 200×0.1+4 = 24ms

  Exception: some DBMS support NULL-inclusive indexes → A4 would work

NULL in joins:
  Standard SQL: NULL ≠ NULL (NULL = anything is NULL, not TRUE)
  A Student with sid=NULL will NOT match any Takes tuple (even one with sid=NULL)
  
  Hash join with NULLs:
    During partition: hash(NULL) → goes to a bucket
    During probe: comparison NULL=NULL → FALSE → no join output
    NULLs safely handled — just no matches produced
    No extra cost; NULLs pass through harmlessly

  Outer joins with NULLs:
    LEFT JOIN: Students with NULL sid still appear in output (with NULL Takes fields)
    Cost: same as regular outer join (Section B, Example 48)
```

> **Key Insight:** Standard B+tree indexes **do not index NULL values**, making `IS NULL` conditions always require a linear scan. Null values in join attributes simply produce no join output in inner joins — the DBMS handles this correctly and efficiently without extra cost.

---

### Example 80: Final Summary — Optimization Decision Tree

**Decision Framework for Any Query:**

```
Given: σ_θ(R) or R ⋈ S

STEP 1: Estimate selectivity
  Calculate RF = 1/V(A,R) for each condition
  Estimate SC = NR × RF
  
STEP 2: For SELECTION — choose algorithm
  IF SC ≤ 3 (very selective):
    → Use index (A2/A3/A4 depending on index type)
  IF SC between 3 and 10% of BR:
    → Compare index vs A1: compute both costs
  IF SC > 10% of BR:
    → Use A1 linear scan (almost always)
  IF conjunctive (AND):
    → Use most selective condition's index (A7)
    → OR composite index if available (A8)
    → OR intersection if all have indexes and combined SC is tiny (A9)
  IF disjunctive (OR):
    → Almost always A1 linear scan
    → A10 only if EVERY condition has index AND combined result is tiny

STEP 3: For JOIN — choose algorithm
  IF both relations tiny (fit in memory): any join = BR+BS
  IF one relation has index on join attr AND SC per probe is tiny (≤3):
    → INLJ (indexed relation inner)
  IF both relations need sorting for other reasons (ORDER BY, etc.):
    → Sort-Merge Join (shared sort)
  IF memory is sufficient (no overflow):
    → Hash Join (usually cheapest: 3×(BR+BS))
  IF memory very limited:
    → BNLJ (predictable degradation)
  DEFAULT:
    → Compute all five join costs; pick minimum

STEP 4: Apply heuristics in order
  1. Push ALL σ as far down as possible (biggest win)
  2. Reorder joins: smallest filtered relation first
  3. Replace × + σ with ⋈
  4. Push ALL π down (reduces record width)
  5. Build left-deep tree for pipelineability
  6. Pipeline unary operators (σ, π) on-the-fly

STEP 5: Enumerate and compare
  For N-way joins: enumerate left-deep trees
  Avoid Cartesian products in plan
  Choose plan with minimum estimated cost b×tT + S×tS
```

**Universal Cost Formula Reminder:**

```
Cost = b × tT + S × tS

Algorithm Quick Reference:
  A1 (scan, non-key):    b=BR,        S=1
  A1 (scan, key):        b=BR/2,      S=1
  A2 (primary, =key):    b=HT+1,      S=HT+1
  A3 (primary, =nonkey): b=HT+b_data, S=HT+1
  A4 (secondary, =key):  b=HT+1,      S=HT+1
  A4 (secondary, nonkey):b=HT+n,      S=HT+n
  NLJ worst:   b=NR_outer×BR_inner+BR_outer,  many S
  BNLJ worst:  b=BR_outer×BR_inner+BR_outer,  fewer S
  SMJ (sorted): b=BR+BS,             S=2
  Hash Join:   b=3×(BR+BS),         S=many
  Ext. Sort:   b=2×BR×passes-BR,    S=many
```

> **Key Insight:** The entire optimization process boils down to three questions: (1) **How many tuples qualify?** (selectivity/SC), (2) **Can we avoid reading irrelevant data?** (index/pushdown), (3) **In what order should we combine results?** (join ordering). Answer these three questions correctly and you will almost always find the optimal or near-optimal plan.

---

## Quick Reference: All 50 Examples

| # | Topic | Key Algorithm | Best Cost |
|---|-------|--------------|-----------|
| 31 | Binary search vs index | A2 vs A1 | A1 wins (9ms) |
| 32 | Break-even: clustering vs scan | A3 | A3 < A1 if SC<59% |
| 33 | A7: choosing right index | A7 (salary, SC=4) | 24.6ms |
| 34 | Range + secondary index disaster | A1 | 24ms (not 3616ms) |
| 35 | Composite index prefix match | A8 vs A1 | A8=16.4ms |
| 36 | Hash index exact match only | Hash index | ~8.2ms |
| 37 | Histogram-based selectivity | RF estimation | A1 always here |
| 38 | A7 vs A8 vs A9 on Supply | A8 composite | 20.5ms |
| 39 | Unindexed conditions (LIKE) | A1 | 8.8ms |
| 40 | GROUP BY — hash aggregation | Hash GROUP BY | 24ms |
| 41 | NLJ vs BNLJ buffer analysis | BNLJ | 148.2ms |
| 42 | INLJ: when index doesn't help | Hash/SMJ | 100.6ms |
| 43 | Sort-merge shared sort (ORDER BY) | SMJ | 132.6ms |
| 44 | Hash join partition overflow | Recursive hash | 195ms |
| 45 | Self-join | Hash self-join | ~90ms |
| 46 | Semi-join (EXISTS) de-correlation | Semi-join | 38ms |
| 47 | Anti-join (NOT IN) | Hash anti-join | 38ms |
| 48 | Outer join | Hash LOJ | 130ms |
| 49 | 3-way join ordering | All orderings same | 220ms |
| 50 | Projection pushdown | π pushdown | 92ms (was 560ms) |
| 51 | Subquery flattening | Join rewrite | 94.6ms (was 24000ms) |
| 52 | Scalar subquery | Eval once | 48ms |
| 53 | UNION optimization | Single scan OR | 18ms |
| 54 | INTERSECT as semi-join | Semi-join | 38ms |
| 55 | HAVING with aggregation | Hash GROUP BY | 24ms |
| 56 | Top-K / ORDER BY LIMIT | Priority queue | 14ms |
| 57 | COUNT(DISTINCT) | Index-only scan | 13.1ms |
| 58 | Star schema join | Filter dims first | 225.7ms |
| 59 | View expansion + pushdown | Inline view | 58.6ms |
| 60 | Covering index | Index-only scan | 34.1ms |
| 61 | Materialized view | MV query = 1 block | 4.1ms |
| 62 | Partition pruning | Scan 1/5 of table | 8ms |
| 63 | Parallel aggregation | Partition-parallel | 8ms |
| 64 | Bushy tree parallelism | Bushy parallel | 234ms |
| 65 | 4-table full optimization | All 6 steps | 237.6ms |
| 66 | Stale statistics impact | Re-analyze needed | 235× slower |
| 67 | Buffer pool allocation | BNLJ chunk size | 292.8ms |
| 68 | Composite sort key | External sort | 80ms |
| 69 | Output write cost | CTAS: include output | +6ms |
| 70 | Adaptive query processing | Mid-query correction | 156ms |
| 71 | Covering index for JOIN | Index-only | 12.4ms |
| 72 | Function-based index | UPPER() disables idx | A1 always |
| 73 | OR-to-UNION rewrite | A1 wins | 24ms |
| 74 | Predicate ordering in scan | Most selective first | CPU opt |
| 75 | Type mismatch in join | Hash join robust | 130ms |
| 76 | Multi-column self-join | Hash join | 320ms |
| 77 | MAX/MIN via index | B+tree rightmost | 8.3ms |
| 78 | Cached plan reuse | Parameter sniffing | Varies |
| 79 | NULL in selection/join | A1 for IS NULL | 24ms |
| 80 | Full decision framework | All techniques | Reference |

---

*CS G516 Advanced Database Systems | BITS Pilani | Dr. Yashvardhan Sharma*
*Examples 31–80 | Classes 6, 7, 8 | Query Processing & Optimization*
