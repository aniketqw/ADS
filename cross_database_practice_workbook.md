# Cross-Database Translation Workbook
## Practice Problems: Same Query, 6 Ways

---

## How to Use This Workbook

For each problem:
1. **Read the requirements** in natural language
2. **Translate to ALL 6 databases**
3. **Check your answers** against the provided solutions
4. **Understand WHY** each database approaches it differently

---

## Problem Set 1: Basic Queries

### Problem 1.1: Find All Instructors in Biology Department

**Requirements**: Return instructor ID, name, and salary for all instructors in the Biology department.

**Your Solutions**:

<details>
<summary>SQL</summary>

```sql
SELECT ID, name, salary
FROM instructor
WHERE dept_name = 'Biology';
```
</details>

<details>
<summary>MongoDB</summary>

```javascript
db.instructors.find(
  { dept_name: "Biology" },
  { ID: 1, name: 1, salary: 1, _id: 0 }
)
```
</details>

<details>
<summary>Cassandra</summary>

```sql
SELECT instructor_id, name, salary
FROM instructors_by_department
WHERE dept_name = 'Biology';
```

**Note**: Requires table with partition key = dept_name
</details>

<details>
<summary>Redis</summary>

```bash
# Get all instructor IDs in Biology
SMEMBERS dept:Biology:instructors

# For each ID, fetch details
HMGET instructor:10101 ID name salary
HMGET instructor:22222 ID name salary
```

**Better approach**: Pipeline or Lua script to batch fetch
</details>

<details>
<summary>Neo4j</summary>

```cypher
MATCH (i:Instructor)-[:WORKS_IN]->(d:Department {dept_name: "Biology"})
RETURN i.ID, i.name, i.salary
```
</details>

<details>
<summary>XQuery</summary>

```xquery
for $instructor in doc("university.xml")//Department[DeptName="Biology"]/Instructors/Instructor
return <Instructor>
  <ID>{$instructor/@id/string()}</ID>
  <n>{$instructor/Name/text()}</n>
  <Salary>{$instructor/Salary/text()}</Salary>
</Instructor>
```
</details>

---

### Problem 1.2: Find All 4-Credit Courses

**Requirements**: List all courses with exactly 4 credits, showing course ID and title.

**Try it yourself, then check solutions**:

<details>
<summary>SQL</summary>

```sql
SELECT course_id, title
FROM course
WHERE credits = 4;
```
</details>

<details>
<summary>MongoDB</summary>

```javascript
db.courses.find(
  { credits: 4 },
  { course_id: 1, title: 1, _id: 0 }
)
```
</details>

<details>
<summary>Cassandra</summary>

```sql
-- Problem: credits is not part of partition key!
-- Option 1: Scan all (with ALLOW FILTERING)
SELECT course_id, title
FROM courses_by_department
WHERE credits = 4
ALLOW FILTERING;

-- Option 2: Create dedicated table
CREATE TABLE courses_by_credits (
    credits INT,
    course_id TEXT,
    title TEXT,
    PRIMARY KEY (credits, course_id)
);

SELECT course_id, title
FROM courses_by_credits
WHERE credits = 4;
```

**Lesson**: In Cassandra, query determines table design!
</details>

<details>
<summary>Redis</summary>

```bash
# Maintain set of courses per credit level
SADD courses:credits:4 "CS-101" "BIO-301" "HIST-201"

# Get all 4-credit courses
SMEMBERS courses:credits:4

# Fetch course details
HMGET course:CS-101 course_id title
```
</details>

<details>
<summary>Neo4j</summary>

```cypher
MATCH (c:Course {credits: 4})
RETURN c.course_id, c.title
```

**Alternative with relationship**:
```cypher
MATCH (c:Course)-[:HAS_CREDITS]->(cr:CreditLevel {value: 4})
RETURN c.course_id, c.title
```
</details>

<details>
<summary>XQuery</summary>

```xquery
for $course in doc("university.xml")//Course[Credits=4]
return <r>
  <CourseID>{$course/CourseID/text()}</CourseID>
  <Title>{$course/Title/text()}</Title>
</r>
```
</details>

---

## Problem Set 2: Aggregation Queries

### Problem 2.1: Average Salary Per Department

**Requirements**: Calculate and display the average instructor salary for each department.

<details>
<summary>SQL</summary>

```sql
SELECT dept_name, AVG(salary) AS avg_salary
FROM instructor
GROUP BY dept_name
ORDER BY avg_salary DESC;
```
</details>

<details>
<summary>MongoDB</summary>

```javascript
db.instructors.aggregate([
  {
    $group: {
      _id: "$dept_name",
      avg_salary: { $avg: "$salary" }
    }
  },
  {
    $sort: { avg_salary: -1 }
  },
  {
    $project: {
      dept_name: "$_id",
      avg_salary: 1,
      _id: 0
    }
  }
])
```
</details>

<details>
<summary>Cassandra</summary>

```sql
-- NOT DIRECTLY POSSIBLE!
-- Aggregation within partition only

-- If you have all instructors in one partition (BAD design):
SELECT dept_name, AVG(salary) AS avg_salary
FROM instructors_single_partition
WHERE partition_key = 'all'
GROUP BY dept_name;

-- Better: Compute in application or use Spark
```

**Reality Check**: Cassandra is NOT designed for ad-hoc aggregations!
</details>

<details>
<summary>Redis</summary>

```bash
# Option 1: Maintain running averages (update on insert)
HSET dept:Biology:stats avg_salary 75000 count 10

# Option 2: Compute in application
# Fetch all instructors, compute avg in code

# Option 3: Use Lua script
EVAL "
  local depts = {}
  local cursor = '0'
  repeat
    local result = redis.call('SCAN', cursor, 'MATCH', 'instructor:*')
    cursor = result[1]
    for _, key in ipairs(result[2]) do
      local dept = redis.call('HGET', key, 'dept_name')
      local salary = tonumber(redis.call('HGET', key, 'salary'))
      if not depts[dept] then
        depts[dept] = { sum = 0, count = 0 }
      end
      depts[dept].sum = depts[dept].sum + salary
      depts[dept].count = depts[dept].count + 1
    end
  until cursor == '0'
  local result = {}
  for dept, stats in pairs(depts) do
    table.insert(result, dept .. ':' .. (stats.sum / stats.count))
  end
  return result
" 0
```
</details>

<details>
<summary>Neo4j</summary>

```cypher
MATCH (i:Instructor)-[:WORKS_IN]->(d:Department)
RETURN d.dept_name AS dept_name, AVG(i.salary) AS avg_salary
ORDER BY avg_salary DESC
```
</details>

<details>
<summary>XQuery</summary>

```xquery
for $dept in doc("university.xml")//Department
let $avg_sal := avg($dept/Instructors/Instructor/Salary)
order by $avg_sal descending
return <DeptAvg>
  <DeptName>{$dept/DeptName/text()}</DeptName>
  <AvgSalary>{$avg_sal}</AvgSalary>
</DeptAvg>
```
</details>

---

### Problem 2.2: Count Students Per Course

**Requirements**: For each course, show how many students are enrolled.

<details>
<summary>SQL</summary>

```sql
SELECT c.course_id, c.title, COUNT(t.ID) AS num_students
FROM course c
LEFT JOIN takes t ON c.course_id = t.course_id
GROUP BY c.course_id, c.title
ORDER BY num_students DESC;
```
</details>

<details>
<summary>MongoDB (Embedded)</summary>

```javascript
// If students are embedded in course document
db.courses.aggregate([
  {
    $project: {
      course_id: 1,
      title: 1,
      num_students: { $size: { $ifNull: ["$enrolled_students", []] } }
    }
  },
  { $sort: { num_students: -1 } }
])
```
</details>

<details>
<summary>MongoDB (Referenced)</summary>

```javascript
db.takes.aggregate([
  {
    $group: {
      _id: "$course_id",
      num_students: { $sum: 1 }
    }
  },
  {
    $lookup: {
      from: "courses",
      localField: "_id",
      foreignField: "course_id",
      as: "course_info"
    }
  },
  { $unwind: "$course_info" },
  {
    $project: {
      course_id: "$_id",
      title: "$course_info.title",
      num_students: 1,
      _id: 0
    }
  },
  { $sort: { num_students: -1 } }
])
```
</details>

<details>
<summary>Cassandra</summary>

```sql
-- Maintain counter table
CREATE TABLE course_enrollment_count (
    course_id TEXT PRIMARY KEY,
    count COUNTER
);

-- Increment on each enrollment
UPDATE course_enrollment_count
SET count = count + 1
WHERE course_id = 'CS-101';

-- Query counts
SELECT course_id, count
FROM course_enrollment_count;

-- Join with course details in application
```
</details>

<details>
<summary>Redis</summary>

```bash
# Maintain count per course
HINCRBY course:CS-101:stats enrollment_count 1

# Get all courses and their counts
SCAN 0 MATCH course:*:stats

# For each course stats hash
HGET course:CS-101:stats enrollment_count
```
</details>

<details>
<summary>Neo4j</summary>

```cypher
MATCH (c:Course)
OPTIONAL MATCH (c)<-[:ENROLLED_IN]-(s:Student)
RETURN c.course_id, c.title, COUNT(s) AS num_students
ORDER BY num_students DESC
```
</details>

<details>
<summary>XQuery</summary>

```xquery
for $course in doc("university.xml")//Course
let $count := count($course/Sections/Section/Students/Student)
order by $count descending
return <r>
  <CourseID>{$course/CourseID/text()}</CourseID>
  <Title>{$course/Title/text()}</Title>
  <NumStudents>{$count}</NumStudents>
</r>
```
</details>

---

## Problem Set 3: Join Queries

### Problem 3.1: Students and Their Advisors

**Requirements**: List each student's name along with their advisor's name.

<details>
<summary>SQL</summary>

```sql
SELECT s.name AS student_name, i.name AS advisor_name
FROM student s
JOIN advisor a ON s.ID = a.s_ID
JOIN instructor i ON a.i_ID = i.ID
ORDER BY s.name;
```
</details>

<details>
<summary>MongoDB (Embedded)</summary>

```javascript
// If advisor info is embedded in student
db.students.find(
  {},
  { name: 1, "advisor.name": 1, _id: 0 }
)
```
</details>

<details>
<summary>MongoDB (Referenced)</summary>

```javascript
db.students.aggregate([
  {
    $lookup: {
      from: "advisors",
      localField: "_id",
      foreignField: "student_id",
      as: "advisor_relation"
    }
  },
  { $unwind: "$advisor_relation" },
  {
    $lookup: {
      from: "instructors",
      localField: "advisor_relation.instructor_id",
      foreignField: "_id",
      as: "advisor"
    }
  },
  { $unwind: "$advisor" },
  {
    $project: {
      student_name: "$name",
      advisor_name: "$advisor.name",
      _id: 0
    }
  },
  { $sort: { student_name: 1 } }
])
```
</details>

<details>
<summary>Cassandra</summary>

```sql
-- Table 1: Students with advisor info denormalized
CREATE TABLE students_with_advisors (
    student_id TEXT PRIMARY KEY,
    student_name TEXT,
    advisor_id TEXT,
    advisor_name TEXT
);

SELECT student_name, advisor_name
FROM students_with_advisors;
```
</details>

<details>
<summary>Redis</summary>

```bash
# Get all student IDs
SMEMBERS students:all

# For each student
HGET student:00128 name           # "Zhang"
HGET student:00128 advisor_id     # "10101"

# Get advisor name
HGET instructor:10101 name        # "Srinivasan"
```
</details>

<details>
<summary>Neo4j</summary>

```cypher
MATCH (s:Student)-[:ADVISED_BY]->(i:Instructor)
RETURN s.name AS student_name, i.name AS advisor_name
ORDER BY student_name
```

**Beautiful!** This is where graphs shine — relationships are first-class.
</details>

<details>
<summary>XQuery</summary>

```xquery
for $student in doc("university.xml")//Student
let $advisor := doc("university.xml")//Instructor[@id=$student/AdvisorID]
order by $student/Name
return <r>
  <StudentName>{$student/Name/text()}</StudentName>
  <AdvisorName>{$advisor/Name/text()}</AdvisorName>
</r>
```
</details>

---

### Problem 3.2: Courses and Their Prerequisites

**Requirements**: Show each course with its prerequisites (course_id and prereq_id).

<details>
<summary>SQL</summary>

```sql
SELECT c.course_id, c.title, p.prereq_id, pc.title AS prereq_title
FROM course c
JOIN prereq p ON c.course_id = p.course_id
JOIN course pc ON p.prereq_id = pc.course_id
ORDER BY c.course_id;
```
</details>

<details>
<summary>MongoDB (Embedded)</summary>

```javascript
// If prerequisites are embedded as array
db.courses.find(
  { prerequisites: { $exists: true, $ne: [] } },
  { course_id: 1, title: 1, prerequisites: 1 }
)
```
</details>

<details>
<summary>MongoDB (Referenced)</summary>

```javascript
db.courses.aggregate([
  {
    $lookup: {
      from: "prerequisites",
      localField: "course_id",
      foreignField: "course_id",
      as: "prereqs"
    }
  },
  { $unwind: "$prereqs" },
  {
    $lookup: {
      from: "courses",
      localField: "prereqs.prereq_id",
      foreignField: "course_id",
      as: "prereq_course"
    }
  },
  { $unwind: "$prereq_course" },
  {
    $project: {
      course_id: 1,
      title: 1,
      prereq_id: "$prereqs.prereq_id",
      prereq_title: "$prereq_course.title"
    }
  }
])
```
</details>

<details>
<summary>Cassandra</summary>

```sql
-- Denormalize prerequisites into course table
CREATE TABLE courses_with_prereqs (
    course_id TEXT,
    prereq_id TEXT,
    course_title TEXT,
    prereq_title TEXT,
    PRIMARY KEY (course_id, prereq_id)
);

SELECT course_id, course_title, prereq_id, prereq_title
FROM courses_with_prereqs;
```
</details>

<details>
<summary>Redis</summary>

```bash
# Store prerequisites as set
SADD course:CS-347:prerequisites "CS-101" "CS-190"

# Get prerequisites
SMEMBERS course:CS-347:prerequisites

# For each prereq, fetch details
HGET course:CS-101 title
```
</details>

<details>
<summary>Neo4j (Graph Perfect Fit!)</summary>

```cypher
MATCH (c:Course)-[:HAS_PREREQ]->(p:Course)
RETURN c.course_id, c.title, p.course_id AS prereq_id, p.title AS prereq_title
ORDER BY c.course_id
```

**Finding all prerequisites recursively**:
```cypher
MATCH path = (c:Course {course_id: "CS-347"})-[:HAS_PREREQ*]->(p:Course)
RETURN c.course_id, p.course_id, length(path) AS depth
```
</details>

<details>
<summary>XQuery</summary>

```xquery
for $course in doc("university.xml")//Course[Prerequisites/Prerequisite]
return <r>
  <CourseID>{$course/CourseID/text()}</CourseID>
  <Title>{$course/Title/text()}</Title>
  <Prerequisites>
  {
    for $prereq in $course/Prerequisites/Prerequisite
    let $prereq_course := doc("university.xml")//Course[CourseID=$prereq/text()]
    return <Prereq>
      <PrereqID>{$prereq/text()}</PrereqID>
      <PrereqTitle>{$prereq_course/Title/text()}</PrereqTitle>
    </Prereq>
  }
  </Prerequisites>
</r>
```
</details>

---

## Problem Set 4: Advanced Queries

### Problem 4.1: Top 3 Highest-Paid Instructors Per Department

<details>
<summary>SQL (Window Function)</summary>

```sql
WITH ranked AS (
  SELECT 
    dept_name,
    name,
    salary,
    RANK() OVER (PARTITION BY dept_name ORDER BY salary DESC) AS salary_rank
  FROM instructor
)
SELECT dept_name, name, salary
FROM ranked
WHERE salary_rank <= 3
ORDER BY dept_name, salary_rank;
```
</details>

<details>
<summary>MongoDB</summary>

```javascript
db.instructors.aggregate([
  { $sort: { dept_name: 1, salary: -1 } },
  {
    $group: {
      _id: "$dept_name",
      top_instructors: {
        $push: { name: "$name", salary: "$salary" }
      }
    }
  },
  {
    $project: {
      dept_name: "$_id",
      top_3: { $slice: ["$top_instructors", 3] },
      _id: 0
    }
  }
])
```
</details>

<details>
<summary>Cassandra</summary>

```sql
-- Create table sorted by salary (clustering key)
CREATE TABLE instructors_by_dept_salary (
    dept_name TEXT,
    salary DECIMAL,
    instructor_id TEXT,
    name TEXT,
    PRIMARY KEY (dept_name, salary, instructor_id)
) WITH CLUSTERING ORDER BY (salary DESC);

-- Query top 3 per department
SELECT name, salary
FROM instructors_by_dept_salary
WHERE dept_name = 'Comp. Sci.'
LIMIT 3;
```

**Must query each department separately!**
</details>

<details>
<summary>Redis</summary>

```bash
# Sorted set per department
ZADD instructors:Comp.Sci.:by_salary 95000 "10101" 85000 "22222" 75000 "33333"

# Get top 3
ZREVRANGE instructors:Comp.Sci.:by_salary 0 2 WITHSCORES
```
</details>

<details>
<summary>Neo4j</summary>

```cypher
MATCH (i:Instructor)-[:WORKS_IN]->(d:Department)
WITH d.dept_name AS dept, COLLECT(i) AS instructors
UNWIND instructors AS instructor
WITH dept, instructor
ORDER BY instructor.salary DESC
RETURN dept, COLLECT({name: instructor.name, salary: instructor.salary})[0..3] AS top_3
```
</details>

<details>
<summary>XQuery</summary>

```xquery
for $dept in doc("university.xml")//Department
let $top_3 := 
  for $instructor in $dept/Instructors/Instructor
  order by $instructor/Salary descending
  return $instructor
return <DeptTop3>
  <DeptName>{$dept/DeptName/text()}</DeptName>
  <TopInstructors>
  {
    for $i in subsequence($top_3, 1, 3)
    return <Instructor>
      <n>{$i/Name/text()}</n>
      <Salary>{$i/Salary/text()}</Salary>
    </Instructor>
  }
  </TopInstructors>
</DeptTop3>
```
</details>

---

### Problem 4.2: Shortest Path Between Two Entities

**Requirements**: Find the shortest connection path from Student "Zhang" to Instructor "Einstein".

This is a **graph problem**! Let's see how different databases handle it.

<details>
<summary>SQL (Recursive CTE - MySQL 8.0+)</summary>

```sql
-- Highly complex! Requires modeling as graph
-- Example: "Zhang takes course taught by Einstein"

WITH RECURSIVE connections AS (
  -- Base: Direct advisor relationship
  SELECT s.ID AS from_id, s.name AS from_name, i.ID AS to_id, i.name AS to_name, 1 AS distance
  FROM student s
  JOIN advisor a ON s.ID = a.s_ID
  JOIN instructor i ON a.i_ID = i.ID
  
  UNION ALL
  
  -- Recursive: Student → Course → Instructor
  SELECT s.ID, s.name, i.ID, i.name, 2
  FROM student s
  JOIN takes t ON s.ID = t.ID
  JOIN teaches te ON t.course_id = te.course_id
  JOIN instructor i ON te.ID = i.ID
)
SELECT *
FROM connections
WHERE from_name = 'Zhang' AND to_name = 'Einstein'
ORDER BY distance
LIMIT 1;
```

**This is PAINFUL in SQL!**
</details>

<details>
<summary>MongoDB</summary>

```javascript
// MongoDB doesn't have native graph traversal
// Must implement BFS in application code or use $graphLookup (limited)

db.students.aggregate([
  { $match: { name: "Zhang" } },
  {
    $graphLookup: {
      from: "takes",
      startWith: "$_id",
      connectFromField: "course_id",
      connectToField: "course_id",
      as: "connections",
      maxDepth: 3
    }
  }
])
```

**Not ideal for graph queries!**
</details>

<details>
<summary>Cassandra</summary>

```
❌ NOT POSSIBLE in Cassandra
Implement in application using multiple queries
```
</details>

<details>
<summary>Redis</summary>

```bash
# Must implement BFS manually
# Store graph as adjacency lists
SADD node:Zhang:neighbors "course:CS-101"
SADD node:CS-101:neighbors "instructor:Einstein"

# Traverse in application code
```
</details>

<details>
<summary>Neo4j (PERFECT FIT!)</summary>

```cypher
MATCH path = shortestPath(
  (s:Student {name: "Zhang"})-[*]-(i:Instructor {name: "Einstein"})
)
RETURN path, length(path) AS distance
```

**One line!** This is why graph databases exist.

**Specific relationship types**:
```cypher
MATCH path = shortestPath(
  (s:Student {name: "Zhang"})-[:ENROLLED_IN|TAUGHT_BY*]-(i:Instructor {name: "Einstein"})
)
RETURN nodes(path), relationships(path)
```
</details>

<details>
<summary>XML</summary>

```
❌ NOT SUITABLE for graph traversal
XML is hierarchical, not graph-based
```
</details>

---

## Summary: Strengths & Weaknesses

### SQL
✅ Complex joins, aggregations, transactions
❌ Rigid schema, vertical scaling limits

### MongoDB
✅ Flexible schema, horizontal scaling, JSON-native
❌ Weak at complex joins, limited transactions (pre-4.0)

### Cassandra
✅ Extreme write throughput, linear scalability
❌ Query-first design, limited ad-hoc queries, eventual consistency

### Redis
✅ Ultra-fast (in-memory), TTL support, pub/sub
❌ Limited query capability, RAM constraints

### Neo4j
✅ Relationship queries, graph algorithms (shortest path, centrality)
❌ Not for tabular data, complex setup

### XML/XQuery
✅ Hierarchical data, validation (DTD/XSD), industry standards
❌ Verbose, slower parsing, not for large datasets

---

## Practice Challenge

**The Ultimate Translation Challenge**: Design a complete university system using ALL 6 databases in tandem!

**Architecture**:
- **SQL**: Core data, transactions (enrollment, grading)
- **MongoDB**: Student profiles, course catalogs (flexible schema)
- **Cassandra**: Audit logs, time-series data (login history)
- **Redis**: Session management, real-time course waitlists
- **Neo4j**: Social network (student connections), prerequisite chains
- **XML**: Data export/import, integration with legacy systems

This is **polyglot persistence** — using the right database for each workload! 🚀
