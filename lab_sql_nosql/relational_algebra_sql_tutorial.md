# Complete Guide: Relational Algebra & SQL
## From Theory to Practice

---

## Table of Contents
1. [What is Relational Algebra?](#what-is-relational-algebra)
2. [Core Relational Algebra Operations](#core-operations)
3. [From Relational Algebra to SQL](#algebra-to-sql)
4. [Step-by-Step Examples](#step-by-step-examples)
5. [Complex Query Building](#complex-queries)
6. [Practice Problems](#practice-problems)

---

## What is Relational Algebra?

**Relational Algebra** is a procedural query language that works on relations (tables). It's the theoretical foundation for SQL and helps you understand:
- **How queries work internally**
- **Query optimization**
- **Database design principles**

### Key Characteristics
- **Closed**: Operations on relations produce relations
- **Procedural**: You specify HOW to get the result (step-by-step)
- **Unordered**: Relations are sets (no inherent order)

---

## Core Relational Algebra Operations

### 1. **Selection (σ)** — Filter Rows
**Symbol**: σ<sub>condition</sub>(R)  
**Purpose**: Selects rows that satisfy a condition  
**Like**: SQL's WHERE clause

**Example**:
```
σ(credits = 3)(course)
```
Reads as: "Select courses where credits equals 3"

**Mathematical Notation**:
```
σ(dept_name = 'Comp. Sci.')(course)
```

**Output**: Only rows from `course` where `dept_name = 'Comp. Sci.'`

---

### 2. **Projection (π)** — Select Columns
**Symbol**: π<sub>attributes</sub>(R)  
**Purpose**: Selects specific columns and eliminates duplicates  
**Like**: SQL's SELECT clause

**Example**:
```
π(title, credits)(course)
```
Reads as: "Project (select) only the title and credits columns from course"

**Mathematical Notation**:
```
π(name, salary)(instructor)
```

**Output**: Only the `name` and `salary` columns (duplicates removed)

---

### 3. **Union (∪)** — Combine Two Relations
**Symbol**: R ∪ S  
**Purpose**: Returns all tuples in R or S (removes duplicates)  
**Requirement**: R and S must be **union-compatible** (same number of attributes with compatible types)

**Example**:
```
π(name)(instructor) ∪ π(name)(student)
```
Reads as: "All names of instructors OR students"

---

### 4. **Set Difference (−)** — Rows in R but not in S
**Symbol**: R − S  
**Purpose**: Returns tuples in R that are NOT in S

**Example**:
```
π(ID)(instructor) − π(ID)(teaches)
```
Reads as: "Instructors who have never taught any course"

---

### 5. **Cartesian Product (×)** — All Combinations
**Symbol**: R × S  
**Purpose**: Combines every row from R with every row from S  
**Result Size**: |R| × |S| rows

**Example**:
```
instructor × department
```
Reads as: "Every instructor paired with every department"  
⚠️ **Warning**: Usually too large to be useful alone; needs selection afterward

---

### 6. **Rename (ρ)** — Rename Relations or Attributes
**Symbol**: ρ<sub>X(A1, A2, ...)</sub>(R)  
**Purpose**: Renames relation R to X and/or renames attributes

**Example**:
```
ρ(teaches_info)(teaches)
```
Reads as: "Rename the teaches relation to teaches_info"

**With Attributes**:
```
ρ(T(instructor_id, crs_id))(teaches)
```
Reads as: "Rename teaches to T and rename its attributes"

---

### 7. **Natural Join (⋈)** — Intelligent Cartesian Product
**Symbol**: R ⋈ S  
**Purpose**: Joins R and S on ALL common attributes (same name + type)  
**Auto-eliminates** duplicate columns

**Example**:
```
instructor ⋈ department
```
Reads as: "Join instructor and department on dept_name"  
(Since `dept_name` is the common attribute)

---

### 8. **Theta Join (⋈<sub>θ</sub>)** — Join with Custom Condition
**Symbol**: R ⋈<sub>θ</sub> S  
**Purpose**: Cartesian product + selection with condition θ  
**Equivalent to**: σ<sub>θ</sub>(R × S)

**Example**:
```
instructor ⋈(instructor.salary > department.budget) department
```
Reads as: "Join where instructor salary exceeds department budget"

---

## From Relational Algebra to SQL

| Relational Algebra | SQL Equivalent |
|-------------------|----------------|
| σ<sub>condition</sub>(R) | `SELECT * FROM R WHERE condition` |
| π<sub>A,B</sub>(R) | `SELECT DISTINCT A, B FROM R` |
| R ∪ S | `SELECT * FROM R UNION SELECT * FROM S` |
| R − S | `SELECT * FROM R EXCEPT SELECT * FROM S` (or NOT IN/NOT EXISTS) |
| R × S | `SELECT * FROM R CROSS JOIN S` (or `FROM R, S`) |
| R ⋈ S | `SELECT * FROM R NATURAL JOIN S` |
| R ⋈<sub>condition</sub> S | `SELECT * FROM R JOIN S ON condition` |
| ρ<sub>X</sub>(R) | `SELECT * FROM R AS X` (alias) |

---

## Step-by-Step Examples

Let's use the **University Schema** from your lab:

```
course(course_id, title, dept_name, credits)
instructor(ID, name, dept_name, salary)
teaches(ID, course_id, sec_id, semester, year)
student(ID, name, dept_name, tot_cred)
takes(ID, course_id, sec_id, semester, year, grade)
```

---

### Example 1: Simple Selection
**Query**: Find all courses in the Computer Science department

#### Step 1: Relational Algebra
```
σ(dept_name = 'Comp. Sci.')(course)
```

#### Step 2: SQL Translation
```sql
SELECT *
FROM course
WHERE dept_name = 'Comp. Sci.';
```

#### Step 3: Add Projection (only course titles)
**Relational Algebra**:
```
π(title)(σ(dept_name = 'Comp. Sci.')(course))
```

**SQL**:
```sql
SELECT title
FROM course
WHERE dept_name = 'Comp. Sci.';
```

---

### Example 2: Combining Selection and Projection
**Query**: Find titles of 3-credit Computer Science courses

#### Step 1: Break Down the Requirements
- **Filter by department**: dept_name = 'Comp. Sci.'
- **Filter by credits**: credits = 3
- **Select only**: title

#### Step 2: Relational Algebra (inside-out)
```
π(title)(σ(dept_name = 'Comp. Sci.' ∧ credits = 3)(course))
```

**Reads as**:
1. From `course`, select rows where dept is CS AND credits is 3
2. From those rows, project only the `title` column

#### Step 3: SQL Translation
```sql
SELECT title
FROM course
WHERE dept_name = 'Comp. Sci.'
  AND credits = 3;
```

---

### Example 3: Natural Join
**Query**: Find instructor names and their department buildings

#### Step 1: Identify Tables Needed
- `instructor(ID, name, dept_name, salary)`
- `department(dept_name, building, budget)`
- **Common attribute**: `dept_name`

#### Step 2: Relational Algebra
```
π(name, building)(instructor ⋈ department)
```

**Reads as**:
1. Natural join `instructor` and `department` on `dept_name`
2. Project `name` and `building`

#### Step 3: SQL Translation (Method 1: NATURAL JOIN)
```sql
SELECT name, building
FROM instructor
NATURAL JOIN department;
```

#### Step 4: SQL Translation (Method 2: Explicit JOIN - Recommended)
```sql
SELECT i.name, d.building
FROM instructor i
JOIN department d ON i.dept_name = d.dept_name;
```

**Why Method 2 is better**: Explicit joins are clearer and prevent accidental joins on unintended columns.

---

### Example 4: Multi-Table Join
**Query**: Find names of students taught by instructor 'Einstein'

#### Step 1: Identify the Path
```
student → takes → teaches → instructor
```

We need:
- `student.name` (target)
- Match `student.ID` = `takes.ID`
- Match `takes.(course_id, sec_id, semester, year)` = `teaches.(course_id, sec_id, semester, year)`
- Match `teaches.ID` = `instructor.ID`
- Filter: `instructor.name = 'Einstein'`

#### Step 2: Relational Algebra
```
π(student.name)(
  σ(instructor.name = 'Einstein')(
    student ⋈ takes ⋈ teaches ⋈ instructor
  )
)
```

#### Step 3: SQL Translation
```sql
SELECT DISTINCT s.name
FROM student s
JOIN takes t ON s.ID = t.ID
JOIN teaches te ON t.course_id = te.course_id
                AND t.sec_id = te.sec_id
                AND t.semester = te.semester
                AND t.year = te.year
JOIN instructor i ON te.ID = i.ID
WHERE i.name = 'Einstein';
```

**Why DISTINCT?**: A student might take multiple courses from Einstein.

---

### Example 5: Set Operations
**Query**: Find all people (instructors OR students) in Computer Science

#### Step 1: Relational Algebra
```
π(name)(σ(dept_name = 'Comp. Sci.')(instructor))
∪
π(name)(σ(dept_name = 'Comp. Sci.')(student))
```

#### Step 2: SQL Translation
```sql
SELECT name
FROM instructor
WHERE dept_name = 'Comp. Sci.'

UNION

SELECT name
FROM student
WHERE dept_name = 'Comp. Sci.';
```

**Note**: `UNION` automatically removes duplicates. Use `UNION ALL` to keep duplicates.

---

### Example 6: Set Difference
**Query**: Find instructors who have NEVER taught a course

#### Step 1: Break Down
- All instructor IDs: `π(ID)(instructor)`
- Instructor IDs who taught: `π(ID)(teaches)`
- Difference: instructors who never taught

#### Step 2: Relational Algebra
```
π(ID)(instructor) − π(ID)(teaches)
```

#### Step 3: SQL Translation (Method 1: NOT IN)
```sql
SELECT ID
FROM instructor
WHERE ID NOT IN (SELECT ID FROM teaches);
```

#### Step 4: SQL Translation (Method 2: LEFT JOIN - Recommended)
```sql
SELECT i.ID, i.name
FROM instructor i
LEFT JOIN teaches t ON i.ID = t.ID
WHERE t.ID IS NULL;
```

**Why Method 2 is better**: Handles NULL values correctly and is often more efficient.

---

### Example 7: Aggregation
**Query**: Find the highest instructor salary

#### Relational Algebra Note
⚠️ **Pure relational algebra has NO built-in aggregation**. Extended versions add:
- ℑ (gamma) for GROUP BY
- Functions like MAX, MIN, AVG, SUM, COUNT

#### Extended Relational Algebra
```
ℑ(MAX(salary) AS max_sal)(instructor)
```

#### SQL Translation
```sql
SELECT MAX(salary) AS max_sal
FROM instructor;
```

---

### Example 8: Subqueries
**Query**: Find all instructors with the highest salary

#### Step 1: Break Down
1. Find max salary: `MAX(salary)`
2. Find instructors whose salary equals that max

#### Step 2: Relational Algebra
```
σ(salary = (ℑ(MAX(salary))(instructor)))(instructor)
```

#### Step 3: SQL Translation
```sql
SELECT name, dept_name, salary
FROM instructor
WHERE salary = (
    SELECT MAX(salary)
    FROM instructor
);
```

---

## Complex Query Building

### Template for Building Queries

1. **Identify what you need** (final output)
2. **List tables involved** (entities)
3. **Find the path** (how tables connect)
4. **Write join conditions** (foreign keys)
5. **Add filters** (WHERE conditions)
6. **Select columns** (projection)

---

### Example 9: Complex Query
**Query**: Find course titles taught in Fall 2017 by instructors from Physics department earning over 80,000

#### Step 1: Identify Requirements
- **Output**: course titles
- **Filters**: 
  - Semester = 'Fall'
  - Year = 2017
  - dept_name = 'Physics'
  - salary > 80000

#### Step 2: Identify Tables
- `course` (for title)
- `section` (for semester/year info)
- `teaches` (connects instructor to course)
- `instructor` (for dept_name, salary)

#### Step 3: Relational Algebra
```
π(title)(
  σ(semester='Fall' ∧ year=2017 ∧ dept_name='Physics' ∧ salary>80000)(
    course ⋈ section ⋈ teaches ⋈ instructor
  )
)
```

#### Step 4: SQL Translation
```sql
SELECT DISTINCT c.title
FROM course c
JOIN section s ON c.course_id = s.course_id
JOIN teaches t ON s.course_id = t.course_id
              AND s.sec_id = t.sec_id
              AND s.semester = t.semester
              AND s.year = t.year
JOIN instructor i ON t.ID = i.ID
WHERE s.semester = 'Fall'
  AND s.year = 2017
  AND i.dept_name = 'Physics'
  AND i.salary > 80000;
```

---

## Practice Problems

### Problem 1: Basic Selection
**Query**: Find all instructors with salary greater than 90,000

<details>
<summary>Click for Answer</summary>

**Relational Algebra**:
```
σ(salary > 90000)(instructor)
```

**SQL**:
```sql
SELECT *
FROM instructor
WHERE salary > 90000;
```
</details>

---

### Problem 2: Projection + Selection
**Query**: Find names of students with more than 100 credits

<details>
<summary>Click for Answer</summary>

**Relational Algebra**:
```
π(name)(σ(tot_cred > 100)(student))
```

**SQL**:
```sql
SELECT name
FROM student
WHERE tot_cred > 100;
```
</details>

---

### Problem 3: Join
**Query**: Find course titles and their department buildings

<details>
<summary>Click for Answer</summary>

**Relational Algebra**:
```
π(title, building)(course ⋈ department)
```

**SQL**:
```sql
SELECT c.title, d.building
FROM course c
JOIN department d ON c.dept_name = d.dept_name;
```
</details>

---

### Problem 4: Multi-Table Join
**Query**: Find student names who took courses in 'Painter' building

<details>
<summary>Click for Answer</summary>

**Relational Algebra**:
```
π(name)(
  σ(building = 'Painter')(
    student ⋈ takes ⋈ section
  )
)
```

**SQL**:
```sql
SELECT DISTINCT s.name
FROM student s
JOIN takes t ON s.ID = t.ID
JOIN section sec ON t.course_id = sec.course_id
                 AND t.sec_id = sec.sec_id
                 AND t.semester = sec.semester
                 AND t.year = sec.year
WHERE sec.building = 'Painter';
```
</details>

---

### Problem 5: Aggregation
**Query**: Count how many instructors are in each department

<details>
<summary>Click for Answer</summary>

**Extended Relational Algebra**:
```
dept_name ℑ(COUNT(ID))(instructor)
```

**SQL**:
```sql
SELECT dept_name, COUNT(*) AS num_instructors
FROM instructor
GROUP BY dept_name;
```
</details>

---

### Problem 6: Set Difference
**Query**: Find students who have never taken any course

<details>
<summary>Click for Answer</summary>

**Relational Algebra**:
```
π(ID)(student) − π(ID)(takes)
```

**SQL Method 1**:
```sql
SELECT ID
FROM student
WHERE ID NOT IN (SELECT ID FROM takes);
```

**SQL Method 2 (Better)**:
```sql
SELECT s.ID, s.name
FROM student s
LEFT JOIN takes t ON s.ID = t.ID
WHERE t.ID IS NULL;
```
</details>

---

### Problem 7: Complex Query
**Query**: Find names of instructors who teach courses with prerequisite 'CS-101'

<details>
<summary>Click for Answer</summary>

**Relational Algebra**:
```
π(name)(
  σ(prereq_id = 'CS-101')(
    instructor ⋈ teaches ⋈ course ⋈ prereq
  )
)
```

**SQL**:
```sql
SELECT DISTINCT i.name
FROM instructor i
JOIN teaches t ON i.ID = t.ID
JOIN course c ON t.course_id = c.course_id
JOIN prereq p ON c.course_id = p.course_id
WHERE p.prereq_id = 'CS-101';
```
</details>

---

## Key Takeaways

### Relational Algebra vs SQL

| Aspect | Relational Algebra | SQL |
|--------|-------------------|-----|
| **Style** | Procedural (how) | Declarative (what) |
| **Duplicates** | Automatically removed | Keep by default (use DISTINCT) |
| **Operations** | Mathematical symbols | English keywords |
| **Purpose** | Theoretical foundation | Practical implementation |

### When to Use Each

**Relational Algebra**:
- Understanding query execution
- Query optimization theory
- Database design validation
- Academic study

**SQL**:
- Real-world database queries
- Application development
- Data analysis
- Database management

---

## Translation Cheat Sheet

```
RELATIONAL ALGEBRA          →    SQL
═══════════════════════════════════════════════════════

σ(condition)(R)             →    WHERE condition

π(A, B)(R)                  →    SELECT DISTINCT A, B

R ⋈ S                       →    R NATURAL JOIN S
                                 (or JOIN ... ON ...)

R ∪ S                       →    UNION

R − S                       →    EXCEPT / NOT IN

R × S                       →    CROSS JOIN

ρ(X)(R)                     →    FROM R AS X

σ(A)(π(B)(R ⋈ S))          →    SELECT DISTINCT B
                                 FROM R JOIN S
                                 WHERE A
```

---

## Summary

1. **Relational Algebra** is the mathematical foundation of databases
2. **Each RA operation** maps cleanly to SQL constructs
3. **Build complex queries** by combining simple operations
4. **Practice translation** between RA and SQL to master both
5. **Use joins** instead of Cartesian products (performance!)
6. **Think step-by-step**: identify data → path → conditions → output

---

## Next Steps

1. Practice converting simple queries back and forth
2. Draw relational algebra expression trees
3. Try optimizing queries by reordering operations
4. Study query execution plans in MySQL
5. Complete all lab exercises using both RA and SQL

**Remember**: Understanding relational algebra makes you a better SQL developer! 🚀
