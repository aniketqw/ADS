# Relational Algebra & SQL Practice Workbook
## Hands-On Exercises with Solutions

---

## University Schema Reference

```
classroom(building, room_number, capacity)
department(dept_name, building, budget)
course(course_id, title, dept_name, credits)
instructor(ID, name, dept_name, salary)
section(course_id, sec_id, semester, year, building, room_number, time_slot_id)
teaches(ID, course_id, sec_id, semester, year)
student(ID, name, dept_name, tot_cred)
takes(ID, course_id, sec_id, semester, year, grade)
advisor(s_ID, i_ID)
time_slot(time_slot_id, day, start_hr, start_min, end_hr, end_min)
prereq(course_id, prereq_id)
```

---

## Section 1: Fundamental Operations

### Exercise 1.1: Selection (σ)
**Task**: Write relational algebra and SQL to find all courses with 4 credits.

**Your Relational Algebra**:
```
σ(credits = 4)(course)
```

**Your SQL**:
```sql
SELECT *
FROM course
WHERE credits = 4;
```

**Explanation**: Selection filters rows based on a condition. Think of it as applying a WHERE clause.

---

### Exercise 1.2: Multiple Conditions
**Task**: Find Biology department courses with 3 credits.

**Your Relational Algebra**:
```
σ(dept_name = 'Biology' ∧ credits = 3)(course)
```

**Your SQL**:
```sql
SELECT *
FROM course
WHERE dept_name = 'Biology'
  AND credits = 3;
```

**Key Point**: ∧ (AND), ∨ (OR), ¬ (NOT) are logical operators in relational algebra.

---

### Exercise 1.3: Projection (π)
**Task**: Get only course IDs and titles from all courses.

**Your Relational Algebra**:
```
π(course_id, title)(course)
```

**Your SQL**:
```sql
SELECT DISTINCT course_id, title
FROM course;
```

**Note**: Relational algebra automatically removes duplicates. In SQL, use DISTINCT to match this behavior (though it's often omitted in practice).

---

### Exercise 1.4: Combining Selection and Projection
**Task**: Get names of instructors earning more than 100,000.

**Your Relational Algebra**:
```
π(name)(σ(salary > 100000)(instructor))
```

**Your SQL**:
```sql
SELECT name
FROM instructor
WHERE salary > 100000;
```

**Reading Tip**: Read relational algebra from inside out:
1. First: σ(salary > 100000)(instructor) → filter instructors
2. Then: π(name)(...) → project only names

---

## Section 2: Join Operations

### Exercise 2.1: Natural Join (⋈)
**Task**: Find instructor names with their department buildings.

**Tables Involved**:
- instructor(ID, name, dept_name, salary)
- department(dept_name, building, budget)
- Common attribute: dept_name

**Your Relational Algebra**:
```
π(name, building)(instructor ⋈ department)
```

**Your SQL (Method 1: Natural Join)**:
```sql
SELECT name, building
FROM instructor
NATURAL JOIN department;
```

**Your SQL (Method 2: Explicit Join - Recommended)**:
```sql
SELECT i.name, d.building
FROM instructor i
JOIN department d ON i.dept_name = d.dept_name;
```

**Why Explicit is Better**: You control exactly which columns to join on, preventing accidental joins.

---

### Exercise 2.2: Multi-Attribute Join
**Task**: Find student names and the courses they're taking.

**Tables Involved**:
- student(ID, name, dept_name, tot_cred)
- takes(ID, course_id, sec_id, semester, year, grade)
- course(course_id, title, dept_name, credits)

**Your Relational Algebra**:
```
π(student.name, title)(student ⋈ takes ⋈ course)
```

**Your SQL**:
```sql
SELECT DISTINCT s.name, c.title
FROM student s
JOIN takes t ON s.ID = t.ID
JOIN course c ON t.course_id = c.course_id;
```

---

### Exercise 2.3: Complex Join with Composite Keys
**Task**: Find names of instructors teaching sections in Fall 2017.

**Key Challenge**: Section needs (course_id, sec_id, semester, year) to match teaches.

**Your Relational Algebra**:
```
π(name)(
  σ(semester='Fall' ∧ year=2017)(
    instructor ⋈ teaches ⋈ section
  )
)
```

**Your SQL**:
```sql
SELECT DISTINCT i.name
FROM instructor i
JOIN teaches t ON i.ID = t.ID
JOIN section s ON t.course_id = s.course_id
               AND t.sec_id = s.sec_id
               AND t.semester = s.semester
               AND t.year = s.year
WHERE s.semester = 'Fall'
  AND s.year = 2017;
```

**Learning Point**: When joining on composite keys, you need multiple ON conditions.

---

### Exercise 2.4: Theta Join (⋈θ)
**Task**: Find instructors whose salary exceeds their department's budget.

**Your Relational Algebra**:
```
π(name, salary, budget)(
  instructor ⋈(instructor.salary > department.budget) department
)
```

**Your SQL**:
```sql
SELECT i.name, i.salary, d.budget
FROM instructor i
JOIN department d ON i.dept_name = d.dept_name
WHERE i.salary > d.budget;
```

**Alternative (More Explicit)**:
```sql
SELECT i.name, i.salary, d.budget
FROM instructor i
JOIN department d ON i.dept_name = d.dept_name
                  AND i.salary > d.budget;
```

---

## Section 3: Set Operations

### Exercise 3.1: Union (∪)
**Task**: Find all people (instructors OR students) in Computer Science department.

**Your Relational Algebra**:
```
π(name)(σ(dept_name='Comp. Sci.')(instructor))
∪
π(name)(σ(dept_name='Comp. Sci.')(student))
```

**Your SQL**:
```sql
SELECT name
FROM instructor
WHERE dept_name = 'Comp. Sci.'

UNION

SELECT name
FROM student
WHERE dept_name = 'Comp. Sci.';
```

**Note**: UNION removes duplicates. Use UNION ALL to keep them.

---

### Exercise 3.2: Set Difference (−)
**Task**: Find courses that exist but have never been taught.

**Your Relational Algebra**:
```
π(course_id)(course) − π(course_id)(teaches)
```

**Your SQL (Method 1: NOT IN)**:
```sql
SELECT course_id
FROM course
WHERE course_id NOT IN (SELECT course_id FROM teaches);
```

**Your SQL (Method 2: LEFT JOIN - Better Performance)**:
```sql
SELECT c.course_id, c.title
FROM course c
LEFT JOIN teaches t ON c.course_id = t.course_id
WHERE t.course_id IS NULL;
```

---

### Exercise 3.3: Intersection
**Task**: Find courses offered by both Comp. Sci. and Physics departments.

**Note**: Pure relational algebra doesn't have ∩ operator, but it can be derived:
```
R ∩ S = R − (R − S)
or
R ∩ S = (R ∪ S) − ((R − S) ∪ (S − R))
```

**Your SQL (Using INTERSECT)**:
```sql
SELECT course_id
FROM course
WHERE dept_name = 'Comp. Sci.'

INTERSECT

SELECT course_id
FROM course
WHERE dept_name = 'Physics';
```

**Your SQL (Using IN - Alternative)**:
```sql
SELECT DISTINCT c1.course_id
FROM course c1
JOIN course c2 ON c1.title = c2.title
WHERE c1.dept_name = 'Comp. Sci.'
  AND c2.dept_name = 'Physics';
```

---

## Section 4: Advanced Queries

### Exercise 4.1: Subquery - Scalar
**Task**: Find all instructors with salary equal to the maximum salary.

**Your Relational Algebra**:
```
σ(salary = MAX(salary))(instructor)
```

**Your SQL**:
```sql
SELECT name, dept_name, salary
FROM instructor
WHERE salary = (SELECT MAX(salary) FROM instructor);
```

**Why it works**: The subquery returns a single value (scalar), which can be used in comparison.

---

### Exercise 4.2: Subquery - Set Membership
**Task**: Find students who have taken courses taught by instructor with ID '10101'.

**Your Relational Algebra**:
```
π(name)(
  student ⋈ (σ(ID='10101')(takes ⋈ teaches))
)
```

**Your SQL (Method 1: IN)**:
```sql
SELECT DISTINCT s.name
FROM student s
WHERE s.ID IN (
    SELECT t.ID
    FROM takes t
    JOIN teaches te ON t.course_id = te.course_id
                    AND t.sec_id = te.sec_id
                    AND t.semester = te.semester
                    AND t.year = te.year
    WHERE te.ID = '10101'
);
```

**Your SQL (Method 2: EXISTS)**:
```sql
SELECT s.name
FROM student s
WHERE EXISTS (
    SELECT 1
    FROM takes t
    JOIN teaches te ON t.course_id = te.course_id
                    AND t.sec_id = te.sec_id
                    AND t.semester = te.semester
                    AND t.year = te.year
    WHERE te.ID = '10101'
      AND t.ID = s.ID
);
```

---

### Exercise 4.3: Aggregation with GROUP BY
**Task**: For each department, count the number of instructors.

**Extended Relational Algebra**:
```
dept_name ℑ COUNT(ID) AS num_instructors (instructor)
```

**Your SQL**:
```sql
SELECT dept_name, COUNT(*) AS num_instructors
FROM instructor
GROUP BY dept_name;
```

---

### Exercise 4.4: HAVING Clause
**Task**: Find departments with more than 3 instructors.

**Extended Relational Algebra**:
```
σ(num_instructors > 3)(
  dept_name ℑ COUNT(ID) AS num_instructors (instructor)
)
```

**Your SQL**:
```sql
SELECT dept_name, COUNT(*) AS num_instructors
FROM instructor
GROUP BY dept_name
HAVING COUNT(*) > 3;
```

**Key Difference**: 
- WHERE filters rows BEFORE grouping
- HAVING filters groups AFTER grouping

---

### Exercise 4.5: Nested Aggregation
**Task**: Find the department with the highest average salary.

**Your SQL (Step by Step)**:

**Step 1**: Calculate average salary per department:
```sql
SELECT dept_name, AVG(salary) AS avg_sal
FROM instructor
GROUP BY dept_name;
```

**Step 2**: Find the maximum of those averages:
```sql
SELECT MAX(avg_sal) AS max_avg
FROM (
    SELECT AVG(salary) AS avg_sal
    FROM instructor
    GROUP BY dept_name
) AS dept_avg;
```

**Step 3**: Find which department has that max average:
```sql
SELECT dept_name, AVG(salary) AS avg_sal
FROM instructor
GROUP BY dept_name
HAVING AVG(salary) = (
    SELECT MAX(avg_sal)
    FROM (
        SELECT AVG(salary) AS avg_sal
        FROM instructor
        GROUP BY dept_name
    ) AS dept_avg
);
```

---

## Section 5: Real-World Scenarios

### Scenario 5.1: Student Transcript
**Task**: Generate a transcript for student with ID '12345' showing:
- Course ID
- Course Title
- Semester
- Year
- Grade

**Your SQL**:
```sql
SELECT 
    t.course_id,
    c.title,
    t.semester,
    t.year,
    t.grade
FROM takes t
JOIN course c ON t.course_id = c.course_id
WHERE t.ID = '12345'
ORDER BY t.year, t.semester, c.title;
```

---

### Scenario 5.2: Course Prerequisites Chain
**Task**: Find all prerequisites (direct and indirect) for course 'CS-347'.

**Note**: This requires recursive queries (not in standard relational algebra).

**Your SQL (Using WITH RECURSIVE - MySQL 8.0+)**:
```sql
WITH RECURSIVE prereq_chain AS (
    -- Base case: direct prerequisites
    SELECT course_id, prereq_id, 1 AS level
    FROM prereq
    WHERE course_id = 'CS-347'
    
    UNION ALL
    
    -- Recursive case: prerequisites of prerequisites
    SELECT p.course_id, p.prereq_id, pc.level + 1
    FROM prereq p
    JOIN prereq_chain pc ON p.course_id = pc.prereq_id
    WHERE pc.level < 5  -- Prevent infinite loops
)
SELECT DISTINCT prereq_id, level
FROM prereq_chain
ORDER BY level, prereq_id;
```

---

### Scenario 5.3: Instructor Workload
**Task**: Find instructors and count how many sections they're teaching in Fall 2017.

**Your SQL**:
```sql
SELECT 
    i.ID,
    i.name,
    COUNT(t.course_id) AS num_sections
FROM instructor i
LEFT JOIN teaches t ON i.ID = t.ID 
                    AND t.semester = 'Fall' 
                    AND t.year = 2017
GROUP BY i.ID, i.name
ORDER BY num_sections DESC;
```

**Why LEFT JOIN**: To include instructors who aren't teaching any sections (they'll have count = 0).

---

### Scenario 5.4: Popular Courses
**Task**: Find courses taken by more than 50 students, showing total enrollment.

**Your SQL**:
```sql
SELECT 
    c.course_id,
    c.title,
    COUNT(DISTINCT t.ID) AS total_students
FROM course c
JOIN takes t ON c.course_id = t.course_id
GROUP BY c.course_id, c.title
HAVING COUNT(DISTINCT t.ID) > 50
ORDER BY total_students DESC;
```

---

## Section 6: Complex Problem Solving

### Problem 6.1: Division Operation
**Task**: Find students who have taken ALL courses offered by the Comp. Sci. department.

**Relational Algebra Concept**: Division (÷)
```
π(ID, course_id)(takes) ÷ π(course_id)(σ(dept_name='Comp. Sci.')(course))
```

**SQL Translation (Method 1: NOT EXISTS)**:
```sql
SELECT s.ID, s.name
FROM student s
WHERE NOT EXISTS (
    -- Courses in CS dept
    SELECT c.course_id
    FROM course c
    WHERE c.dept_name = 'Comp. Sci.'
    
    AND NOT EXISTS (
        -- Courses the student has taken
        SELECT t.course_id
        FROM takes t
        WHERE t.ID = s.ID
          AND t.course_id = c.course_id
    )
);
```

**SQL Translation (Method 2: GROUP BY + HAVING)**:
```sql
SELECT s.ID, s.name
FROM student s
WHERE (
    SELECT COUNT(DISTINCT t.course_id)
    FROM takes t
    JOIN course c ON t.course_id = c.course_id
    WHERE t.ID = s.ID
      AND c.dept_name = 'Comp. Sci.'
) = (
    SELECT COUNT(*)
    FROM course
    WHERE dept_name = 'Comp. Sci.'
);
```

---

### Problem 6.2: Ranking
**Task**: Rank instructors by salary within each department.

**Your SQL (Using Window Functions - MySQL 8.0+)**:
```sql
SELECT 
    dept_name,
    name,
    salary,
    RANK() OVER (PARTITION BY dept_name ORDER BY salary DESC) AS salary_rank
FROM instructor
ORDER BY dept_name, salary_rank;
```

---

### Problem 6.3: Self-Join
**Task**: Find pairs of instructors who work in the same department.

**Your Relational Algebra**:
```
σ(i1.dept_name = i2.dept_name ∧ i1.ID < i2.ID)(
  ρ(i1)(instructor) × ρ(i2)(instructor)
)
```

**Your SQL**:
```sql
SELECT 
    i1.name AS instructor1,
    i2.name AS instructor2,
    i1.dept_name
FROM instructor i1
JOIN instructor i2 ON i1.dept_name = i2.dept_name
                   AND i1.ID < i2.ID
ORDER BY i1.dept_name, i1.name;
```

**Why i1.ID < i2.ID**: Prevents both (A,B) and (B,A) pairs, and avoids pairing instructor with themselves.

---

## Section 7: Query Optimization Insights

### Optimization 7.1: Push Selection Down
**Bad**:
```
π(title)(σ(credits = 3)(course ⋈ department))
```

**Good**:
```
π(title)(σ(credits = 3)(course) ⋈ department)
```

**Why**: Filter early to reduce intermediate result size.

**SQL Impact**:
```sql
-- Less efficient (filters after join)
SELECT c.title
FROM course c
JOIN department d ON c.dept_name = d.dept_name
WHERE c.credits = 3;

-- More efficient (filter may help optimizer)
SELECT c.title
FROM (SELECT * FROM course WHERE credits = 3) c
JOIN department d ON c.dept_name = d.dept_name;
```

---

### Optimization 7.2: Avoid Cartesian Product
**Bad**:
```
σ(instructor.dept_name = department.dept_name)(
  instructor × department
)
```

**Good**:
```
instructor ⋈ department
```

**Why**: Natural join is much faster than filtering a Cartesian product.

---

## Section 8: Practice Test

### Question 1
Find the names of all students who have never taken a course.

**Your Answer**:
<details>
<summary>Click for solution</summary>

**Relational Algebra**:
```
π(name)(student) − π(name)(student ⋈ takes)
```

Or:
```
π(name)(σ(ID NOT IN π(ID)(takes))(student))
```

**SQL**:
```sql
SELECT s.name
FROM student s
LEFT JOIN takes t ON s.ID = t.ID
WHERE t.ID IS NULL;
```
</details>

---

### Question 2
Find courses that have prerequisites but have never been taken by any student.

**Your Answer**:
<details>
<summary>Click for solution</summary>

**Relational Algebra**:
```
π(course_id)(prereq) − π(course_id)(takes)
```

**SQL**:
```sql
SELECT DISTINCT p.course_id, c.title
FROM prereq p
JOIN course c ON p.course_id = c.course_id
WHERE p.course_id NOT IN (
    SELECT DISTINCT course_id FROM takes
);
```
</details>

---

### Question 3
Find departments where the average instructor salary is greater than $70,000.

**Your Answer**:
<details>
<summary>Click for solution</summary>

**Extended Relational Algebra**:
```
σ(avg_sal > 70000)(
  dept_name ℑ AVG(salary) AS avg_sal (instructor)
)
```

**SQL**:
```sql
SELECT dept_name, AVG(salary) AS avg_sal
FROM instructor
GROUP BY dept_name
HAVING AVG(salary) > 70000;
```
</details>

---

### Question 4
Find students who have taken at least one course taught by their own advisor.

**Your Answer**:
<details>
<summary>Click for solution</summary>

**Relational Algebra**:
```
π(student.name)(
  student ⋈ advisor ⋈ teaches ⋈ takes
  WHERE advisor.i_ID = teaches.ID
    AND takes.ID = advisor.s_ID
)
```

**SQL**:
```sql
SELECT DISTINCT s.name
FROM student s
JOIN advisor a ON s.ID = a.s_ID
JOIN teaches te ON a.i_ID = te.ID
JOIN takes tk ON s.ID = tk.ID
             AND te.course_id = tk.course_id
             AND te.sec_id = tk.sec_id
             AND te.semester = tk.semester
             AND te.year = tk.year;
```
</details>

---

## Quick Reference Card

### Relational Algebra Symbols
```
σ   Selection (filter rows)
π   Projection (select columns)
⋈   Natural Join
×   Cartesian Product
∪   Union
−   Set Difference
ρ   Rename
ℑ   Aggregation/Grouping
∧   AND
∨   OR
¬   NOT
```

### SQL Equivalents
```
σ        →  WHERE
π        →  SELECT DISTINCT
⋈        →  JOIN ... ON
×        →  CROSS JOIN
∪        →  UNION
−        →  EXCEPT / NOT IN / LEFT JOIN ... IS NULL
ρ        →  AS (alias)
ℑ        →  GROUP BY + aggregates
```

---

## Summary Checklist

✅ **Understand the 8 core operations**
✅ **Know how to translate RA to SQL**
✅ **Master join types and when to use each**
✅ **Practice set operations (UNION, EXCEPT)**
✅ **Learn subquery patterns (IN, EXISTS, scalar)**
✅ **Use aggregation with GROUP BY and HAVING**
✅ **Optimize queries by pushing selections down**
✅ **Solve complex problems step-by-step**

**Next Steps**: Go through your lab exercises and write both RA and SQL for each!
