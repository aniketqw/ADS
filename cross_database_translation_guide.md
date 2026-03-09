# Cross-Database Translation Guide
## From Relational Algebra & SQL to NoSQL & XML

**A Complete Bridge Between Database Paradigms**

---

## 📚 Table of Contents

1. [The Paradigm Shift — Understanding the Differences](#paradigm-shift)
2. [Data Modeling Across Databases](#data-modeling)
3. [Query Translation Matrix](#query-matrix)
4. [Detailed Examples: Same Query, 6 Ways](#detailed-examples)
5. [Complex Queries Across Paradigms](#complex-queries)
6. [Decision Tree: Which Database When?](#decision-tree)

---

## The Paradigm Shift

### Understanding the Fundamental Differences

Before we translate queries, we must understand how each database **thinks** about data:

| Database | Data Model | Query Style | Strength | Trade-off |
|----------|-----------|-------------|----------|-----------|
| **SQL** | Tables + Rows | Declarative (what you want) | Complex queries, ACID | Vertical scaling, rigid schema |
| **MongoDB** | Documents (JSON) | Aggregation pipelines | Flexible schema, horizontal scaling | No complex joins |
| **Cassandra** | Wide columns | CQL (query-first design) | Write-heavy, massive scale | Limited queries, eventual consistency |
| **Redis** | Key-Value pairs | Direct access | Ultra-fast (in-memory) | Limited query capability |
| **Neo4j** | Graph (nodes + edges) | Pattern matching (Cypher) | Relationship traversal | Not for tabular data |
| **XML/XQuery** | Hierarchical tree | Path navigation | Nested data, validation | Verbose, slower parsing |

### The Mental Model Shift

**SQL Thinking**: "I have normalized tables and will JOIN them to answer queries"
**MongoDB Thinking**: "I have flexible documents and will embed or link them based on access patterns"
**Cassandra Thinking**: "What queries will I run? Let me design tables FOR those queries"
**Redis Thinking**: "What's my key? How fast can I get/set this value?"
**Neo4j Thinking**: "What relationships exist? How do nodes connect?"
**XML Thinking**: "How is this data nested? What's the path to what I need?"

---

## Data Modeling Across Databases

Let's take the **University Schema** and model it in each database.

### Original SQL Schema

```sql
-- Relational (normalized)
course(course_id, title, dept_name, credits)
instructor(ID, name, dept_name, salary)
teaches(ID, course_id, sec_id, semester, year)
student(ID, name, dept_name, tot_cred)
takes(ID, course_id, sec_id, semester, year, grade)
department(dept_name, building, budget)
```

**Key Relationships**:
- Instructor → Department (many-to-one)
- Course → Department (many-to-one)
- Instructor → Teaches → Course (many-to-many)
- Student → Takes → Course (many-to-many)

---

### MongoDB Schema (Document Model)

**Approach 1: Embedding (Denormalized)**
```javascript
// Collection: courses
{
  _id: "CS-101",
  title: "Intro to Computer Science",
  dept_name: "Comp. Sci.",
  credits: 3,
  
  // Embed department info (denormalized)
  department: {
    building: "Taylor",
    budget: 100000
  },
  
  // Embed instructors teaching this course
  instructors: [
    {
      id: "10101",
      name: "Srinivasan",
      salary: 65000,
      sections: [
        { sec_id: "1", semester: "Fall", year: 2017 }
      ]
    }
  ],
  
  // Embed enrolled students
  enrolled_students: [
    {
      id: "00128",
      name: "Zhang",
      grade: "A"
    }
  ]
}
```

**Approach 2: References (Normalized)**
```javascript
// Collection: courses
{
  _id: "CS-101",
  title: "Intro to Computer Science",
  dept_name: "Comp. Sci.",
  credits: 3
}

// Collection: instructors
{
  _id: "10101",
  name: "Srinivasan",
  dept_name: "Comp. Sci.",
  salary: 65000
}

// Collection: teaches (junction)
{
  instructor_id: "10101",
  course_id: "CS-101",
  sec_id: "1",
  semester: "Fall",
  year: 2017
}
```

**Design Decision**:
- Use **embedding** when: data is always accessed together, 1-to-few relationships
- Use **references** when: data changes independently, many-to-many, data reuse

---

### Cassandra Schema (Query-First Design)

**Rule**: One table per query pattern!

```sql
-- Query 1: "Find all courses in a department"
CREATE TABLE courses_by_department (
    dept_name TEXT,
    course_id TEXT,
    title TEXT,
    credits INT,
    PRIMARY KEY (dept_name, course_id)
);

-- Query 2: "Find all courses taught by an instructor"
CREATE TABLE courses_by_instructor (
    instructor_id TEXT,
    course_id TEXT,
    semester TEXT,
    year INT,
    title TEXT,
    PRIMARY KEY (instructor_id, course_id, semester, year)
);

-- Query 3: "Find students enrolled in a course"
CREATE TABLE students_by_course (
    course_id TEXT,
    sec_id TEXT,
    semester TEXT,
    year INT,
    student_id TEXT,
    student_name TEXT,
    grade TEXT,
    PRIMARY KEY ((course_id, sec_id, semester, year), student_id)
);
```

**Why duplicate data?**: Cassandra optimizes for read speed. Data duplication is intentional and necessary.

**Partition Key**: Determines which node stores the data (must be in every query)
**Clustering Key**: Determines sort order within partition

---

### Redis Schema (Key-Value Design)

**Approach**: Use **key naming conventions** to simulate structure

```bash
# Courses (Hash)
HSET course:CS-101 title "Intro to CS" dept_name "Comp. Sci." credits 3

# Instructors (Hash)
HSET instructor:10101 name "Srinivasan" dept_name "Comp. Sci." salary 65000

# Department to courses mapping (Set)
SADD dept:Comp.Sci.:courses "CS-101" "CS-190" "CS-315"

# Course enrollments (Sorted Set - sorted by grade)
ZADD course:CS-101:students 4.0 "student:00128"  # 4.0 = A
ZADD course:CS-101:students 3.7 "student:12345"  # 3.7 = A-

# Teaching assignments (Set)
SADD instructor:10101:courses "CS-101" "CS-347"

# Instructor leaderboard (Sorted Set by salary)
ZADD instructors:by_salary 65000 "10101" 95000 "10211"
```

**Key Pattern**: `object_type:identifier:attribute` or `object_type:identifier`

**When to use each structure**:
- **Hash**: Object with multiple fields
- **Set**: Unordered collection (e.g., courses in a department)
- **Sorted Set**: Ranked/ordered data (leaderboards, top students)
- **String**: Simple values, counters

---

### Neo4j Schema (Graph Model)

**Approach**: Entities become nodes, relationships become edges

```cypher
// Nodes (entities)
CREATE (cs:Department {dept_name: "Comp. Sci.", building: "Taylor", budget: 100000})
CREATE (i:Instructor {id: "10101", name: "Srinivasan", salary: 65000})
CREATE (c:Course {course_id: "CS-101", title: "Intro to CS", credits: 3})
CREATE (s:Student {id: "00128", name: "Zhang", tot_cred: 102})

// Relationships (connections)
CREATE (i)-[:WORKS_IN]->(cs)
CREATE (c)-[:OFFERED_BY]->(cs)
CREATE (i)-[:TEACHES {sec_id: "1", semester: "Fall", year: 2017}]->(c)
CREATE (s)-[:ENROLLED_IN {semester: "Fall", year: 2017, grade: "A"}]->(c)
```

**Graph Structure**:
```
(Instructor)-[:TEACHES]->(Course)-[:OFFERED_BY]->(Department)
     ^                                                   ^
     |                                                   |
     └──────────────[:WORKS_IN]─────────────────────────┘

(Student)-[:ENROLLED_IN]->(Course)
```

**Key Insight**: Relationships are **first-class citizens** — as important as nodes!

---

### XML Schema (Hierarchical/Tree Model)

**Approach**: Nested structure based on containment

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE University SYSTEM "university.dtd">
<University>
  <Departments>
    <Department>
      <DeptName>Comp. Sci.</DeptName>
      <Building>Taylor</Building>
      <Budget>100000</Budget>
      
      <Courses>
        <Course>
          <CourseID>CS-101</CourseID>
          <Title>Intro to Computer Science</Title>
          <Credits>3</Credits>
          
          <Sections>
            <Section semester="Fall" year="2017" sec_id="1">
              <Instructor id="10101" name="Srinivasan"/>
              <Students>
                <Student id="00128" name="Zhang" grade="A"/>
              </Students>
            </Section>
          </Sections>
        </Course>
      </Courses>
      
      <Instructors>
        <Instructor id="10101">
          <Name>Srinivasan</Name>
          <Salary>65000</Salary>
        </Instructor>
      </Instructors>
    </Department>
  </Departments>
</University>
```

**DTD Definition**:
```dtd
<!ELEMENT University (Departments)>
<!ELEMENT Departments (Department+)>
<!ELEMENT Department (DeptName, Building, Budget, Courses, Instructors)>
<!ELEMENT Course (CourseID, Title, Credits, Sections)>
<!ELEMENT Section (Instructor, Students)>
<!ATTLIST Section semester CDATA #REQUIRED year CDATA #REQUIRED sec_id CDATA #REQUIRED>
```

---

## Query Translation Matrix

### Query 1: Find All Courses in Computer Science Department

#### SQL (Relational)
```sql
SELECT course_id, title, credits
FROM course
WHERE dept_name = 'Comp. Sci.';
```

#### Relational Algebra
```
π(course_id, title, credits)(σ(dept_name='Comp. Sci.')(course))
```

#### MongoDB
```javascript
db.courses.find(
  { dept_name: "Comp. Sci." },
  { course_id: 1, title: 1, credits: 1, _id: 0 }
)
```

#### Cassandra
```sql
-- Requires table designed for this query
SELECT course_id, title, credits
FROM courses_by_department
WHERE dept_name = 'Comp. Sci.';
```

#### Redis
```bash
# First, get all courses in department
SMEMBERS dept:Comp.Sci.:courses

# Then fetch each course details
HGETALL course:CS-101
HGETALL course:CS-190
# ... for each returned course_id
```

#### Neo4j (Cypher)
```cypher
MATCH (c:Course)-[:OFFERED_BY]->(d:Department {dept_name: "Comp. Sci."})
RETURN c.course_id, c.title, c.credits
```

#### XPath
```xpath
/University/Departments/Department[DeptName='Comp. Sci.']/Courses/Course
```

#### XQuery
```xquery
for $course in doc("university.xml")//Department[DeptName="Comp. Sci."]/Courses/Course
return <Result>
  <CourseID>{$course/CourseID/text()}</CourseID>
  <Title>{$course/Title/text()}</Title>
  <Credits>{$course/Credits/text()}</Credits>
</Result>
```

---

### Query 2: Count Instructors Per Department

#### SQL
```sql
SELECT dept_name, COUNT(*) AS num_instructors
FROM instructor
GROUP BY dept_name;
```

#### Relational Algebra (Extended)
```
dept_name ℑ COUNT(*) AS num_instructors (instructor)
```

#### MongoDB
```javascript
db.instructors.aggregate([
  {
    $group: {
      _id: "$dept_name",
      num_instructors: { $sum: 1 }
    }
  },
  {
    $project: {
      dept_name: "$_id",
      num_instructors: 1,
      _id: 0
    }
  }
])
```

#### Cassandra
```sql
-- NOT POSSIBLE with standard query!
-- Would need to:
-- 1. Read all instructors
-- 2. Group in application code
-- OR create a counter table

CREATE TABLE instructor_count_by_dept (
    dept_name TEXT PRIMARY KEY,
    count COUNTER
);

-- Update on each insert/delete
UPDATE instructor_count_by_dept
SET count = count + 1
WHERE dept_name = 'Comp. Sci.';
```

#### Redis
```bash
# Increment counter on each insert
HINCRBY dept:stats:instructors "Comp. Sci." 1

# Get all counts
HGETALL dept:stats:instructors
```

#### Neo4j
```cypher
MATCH (i:Instructor)-[:WORKS_IN]->(d:Department)
RETURN d.dept_name AS dept_name, COUNT(i) AS num_instructors
ORDER BY num_instructors DESC
```

#### XQuery
```xquery
for $dept in doc("university.xml")//Department
let $count := count($dept/Instructors/Instructor)
return <DeptCount>
  <DeptName>{$dept/DeptName/text()}</DeptName>
  <InstructorCount>{$count}</InstructorCount>
</DeptCount>
```

---

### Query 3: Find Students Taught by Instructor 'Srinivasan'

#### SQL
```sql
SELECT DISTINCT s.ID, s.name
FROM student s
JOIN takes t ON s.ID = t.ID
JOIN teaches te ON t.course_id = te.course_id
                AND t.sec_id = te.sec_id
                AND t.semester = te.semester
                AND t.year = te.year
JOIN instructor i ON te.ID = i.ID
WHERE i.name = 'Srinivasan';
```

#### MongoDB (with $lookup)
```javascript
db.students.aggregate([
  {
    $lookup: {
      from: "takes",
      localField: "_id",
      foreignField: "student_id",
      as: "enrollments"
    }
  },
  { $unwind: "$enrollments" },
  {
    $lookup: {
      from: "teaches",
      let: {
        cid: "$enrollments.course_id",
        sid: "$enrollments.sec_id",
        sem: "$enrollments.semester",
        yr: "$enrollments.year"
      },
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [
                { $eq: ["$course_id", "$$cid"] },
                { $eq: ["$sec_id", "$$sid"] },
                { $eq: ["$semester", "$$sem"] },
                { $eq: ["$year", "$$yr"] }
              ]
            }
          }
        }
      ],
      as: "teaching"
    }
  },
  { $unwind: "$teaching" },
  {
    $lookup: {
      from: "instructors",
      localField: "teaching.instructor_id",
      foreignField: "_id",
      as: "instructor"
    }
  },
  { $unwind: "$instructor" },
  { $match: { "instructor.name": "Srinivasan" } },
  {
    $project: {
      _id: 1,
      name: 1
    }
  }
])
```

**Note**: This is complex! MongoDB doesn't shine at multi-hop joins. Better approach: embed teaching info in student docs if this is a common query.

#### Cassandra
```sql
-- Query 1: Find courses taught by Srinivasan
SELECT course_id, sec_id, semester, year
FROM courses_by_instructor
WHERE instructor_id = '10101';  -- Assuming we know ID from name

-- Query 2: For each course, find students
-- (Requires multiple queries)
SELECT student_id, student_name
FROM students_by_course
WHERE course_id = 'CS-101'
  AND sec_id = '1'
  AND semester = 'Fall'
  AND year = 2017;
```

**Reality**: Multi-step query executed in application code.

#### Redis
```bash
# 1. Find instructor ID by name (requires scanning or maintaining index)
HGET instructor:name:Srinivasan id  # Returns "10101"

# 2. Get courses taught by this instructor
SMEMBERS instructor:10101:teaches  # Returns set of "course:CS-101:1:Fall:2017"

# 3. For each course section, get students
SMEMBERS section:CS-101:1:Fall:2017:students

# 4. For each student ID, get details
HGETALL student:00128
```

#### Neo4j (Graph Shines Here!)
```cypher
MATCH (s:Student)-[:ENROLLED_IN]->(c:Course)<-[:TEACHES]-(i:Instructor {name: "Srinivasan"})
RETURN DISTINCT s.id, s.name
```

**Note**: This is where graph databases excel — multi-hop relationships are natural!

#### XQuery
```xquery
for $student in doc("university.xml")//Student
where $student/Sections/Section/Instructor[@name="Srinivasan"]
return <Student>
  <ID>{$student/@id}</ID>
  <Name>{$student/Name/text()}</Name>
</Student>
```

---

## Detailed Examples: Same Query, 6 Ways

Let's deep-dive into more complex scenarios.

---

### Example 1: Find Instructors with Salary > 80,000 in Physics Department

#### Relational Algebra
```
π(name, salary)(σ(dept_name='Physics' ∧ salary>80000)(instructor))
```

#### SQL
```sql
SELECT name, salary
FROM instructor
WHERE dept_name = 'Physics'
  AND salary > 80000;
```

#### MongoDB
```javascript
db.instructors.find(
  {
    dept_name: "Physics",
    salary: { $gt: 80000 }
  },
  {
    name: 1,
    salary: 1,
    _id: 0
  }
)
```

**Explanation**:
- `$gt: 80000` is MongoDB's "greater than" operator
- Projection `{ name: 1, salary: 1, _id: 0 }` includes name and salary, excludes _id

#### Cassandra
```sql
-- Requires table designed for this query pattern
SELECT name, salary
FROM instructors_by_department
WHERE dept_name = 'Physics'
  AND salary > 80000
ALLOW FILTERING;
```

**Warning**: `ALLOW FILTERING` scans all partitions = slow! Better design:

```sql
-- Materialized view or separate table
CREATE TABLE high_salary_instructors (
    salary_range TEXT,  -- e.g., "80k-100k"
    dept_name TEXT,
    instructor_id TEXT,
    name TEXT,
    salary DECIMAL,
    PRIMARY KEY ((salary_range), dept_name, instructor_id)
);
```

#### Redis
```bash
# Option 1: Scan all instructor hashes (inefficient)
SCAN 0 MATCH instructor:* COUNT 100

# For each instructor:
HMGET instructor:10101 name dept_name salary
# Filter in application code

# Option 2: Maintain sorted set by salary
ZRANGEBYSCORE instructors:Physics:by_salary 80000 +inf WITHSCORES
```

**Better Design**: Use sorted sets grouped by department

```bash
ZADD instructors:Physics:by_salary 95000 "10101"
ZADD instructors:Physics:by_salary 85000 "22222"

# Query
ZRANGEBYSCORE instructors:Physics:by_salary 80000 +inf
```

#### Neo4j
```cypher
MATCH (i:Instructor)-[:WORKS_IN]->(d:Department {dept_name: "Physics"})
WHERE i.salary > 80000
RETURN i.name, i.salary
```

#### XQuery
```xquery
for $instructor in doc("university.xml")//Department[DeptName="Physics"]/Instructors/Instructor
where $instructor/Salary > 80000
return <Result>
  <Name>{$instructor/Name/text()}</Name>
  <Salary>{$instructor/Salary/text()}</Salary>
</Result>
```

---

### Example 2: Find Course Title and Department Building

#### SQL (with JOIN)
```sql
SELECT c.title, d.building
FROM course c
JOIN department d ON c.dept_name = d.dept_name;
```

#### Relational Algebra
```
π(title, building)(course ⋈ department)
```

#### MongoDB (Option 1: $lookup)
```javascript
db.courses.aggregate([
  {
    $lookup: {
      from: "departments",
      localField: "dept_name",
      foreignField: "_id",
      as: "dept_info"
    }
  },
  { $unwind: "$dept_info" },
  {
    $project: {
      title: 1,
      building: "$dept_info.building",
      _id: 0
    }
  }
])
```

**MongoDB (Option 2: Embedding - Better!)**
```javascript
// If building doesn't change often, embed it in course document
db.courses.find(
  {},
  { title: 1, "department.building": 1, _id: 0 }
)
```

**Design Lesson**: For 1-to-1 or 1-to-few that's read together → embed!

#### Cassandra
```sql
-- Must denormalize at insert time
CREATE TABLE courses_with_building (
    dept_name TEXT,
    course_id TEXT,
    title TEXT,
    building TEXT,  -- Denormalized from department
    PRIMARY KEY (dept_name, course_id)
);

SELECT title, building
FROM courses_with_building
WHERE dept_name = 'Comp. Sci.';
```

#### Redis
```bash
# For each course, fetch building separately
HGET course:CS-101 dept_name     # Returns "Comp. Sci."
HGET dept:Comp.Sci. building     # Returns "Taylor"

# Or embed in course hash at creation
HSET course:CS-101 title "Intro to CS" building "Taylor"
```

#### Neo4j
```cypher
MATCH (c:Course)-[:OFFERED_BY]->(d:Department)
RETURN c.title, d.building
```

**Graph Advantage**: Relationship traversal is O(1), no expensive joins!

#### XPath
```xpath
/University/Departments/Department/Courses/Course/
  (Title | ../../Building)
```

#### XQuery
```xquery
for $course in doc("university.xml")//Course
let $dept := $course/ancestor::Department
return <Result>
  <Title>{$course/Title/text()}</Title>
  <Building>{$dept/Building/text()}</Building>
</Result>
```

---

### Example 3: Highest Salary and Who Earns It

#### SQL (with Subquery)
```sql
SELECT name, salary
FROM instructor
WHERE salary = (SELECT MAX(salary) FROM instructor);
```

#### MongoDB
```javascript
// Step 1: Find max salary
db.instructors.aggregate([
  { $group: { _id: null, max_sal: { $max: "$salary" } } }
])

// Step 2: Find instructors with that salary
// (Or combine in one pipeline)
db.instructors.aggregate([
  { $sort: { salary: -1 } },
  { $limit: 1 },
  { $project: { name: 1, salary: 1, _id: 0 } }
])
```

**Alternative (handles ties)**:
```javascript
db.instructors.aggregate([
  {
    $group: {
      _id: null,
      max_salary: { $max: "$salary" },
      instructors: { $push: "$$ROOT" }
    }
  },
  { $unwind: "$instructors" },
  { $match: { $expr: { $eq: ["$instructors.salary", "$max_salary"] } } },
  { $project: { name: "$instructors.name", salary: "$instructors.salary" } }
])
```

#### Cassandra
```sql
-- Not directly possible! Must:
-- 1. Read all instructors
-- 2. Find max in application code

-- OR maintain a sorted leaderboard
SELECT instructor_id, name, salary
FROM instructor_leaderboard
LIMIT 1;
```

#### Redis
```bash
# Maintain sorted set by salary
ZADD instructor:leaderboard 65000 "10101" 95000 "22222" 85000 "33333"

# Get top earner
ZREVRANGE instructor:leaderboard 0 0 WITHSCORES
# Returns: 1) "22222" 2) "95000"

# Fetch name
HGET instructor:22222 name
```

#### Neo4j
```cypher
MATCH (i:Instructor)
RETURN i.name, i.salary
ORDER BY i.salary DESC
LIMIT 1
```

**Handling ties**:
```cypher
MATCH (i:Instructor)
WITH MAX(i.salary) AS max_sal
MATCH (i2:Instructor)
WHERE i2.salary = max_sal
RETURN i2.name, i2.salary
```

#### XQuery
```xquery
let $max_salary := max(doc("university.xml")//Instructor/Salary)
for $instructor in doc("university.xml")//Instructor
where $instructor/Salary = $max_salary
return <HighestPaid>
  <Name>{$instructor/Name/text()}</Name>
  <Salary>{$instructor/Salary/text()}</Salary>
</HighestPaid>
```

---

## Complex Queries Across Paradigms

### Multi-Table Join: Students Enrolled in Courses Taught in 'Watson' Building

#### SQL
```sql
SELECT DISTINCT s.name
FROM student s
JOIN takes t ON s.ID = t.ID
JOIN section sec ON t.course_id = sec.course_id
                 AND t.sec_id = sec.sec_id
                 AND t.semester = sec.semester
                 AND t.year = sec.year
WHERE sec.building = 'Watson';
```

#### MongoDB (Embedded Approach - Best)
```javascript
// If section info is embedded in student enrollment
db.students.aggregate([
  { $unwind: "$enrollments" },
  { $match: { "enrollments.building": "Watson" } },
  { $project: { name: 1, _id: 0 } }
])
```

#### MongoDB (Reference Approach - Requires Multiple Lookups)
```javascript
db.students.aggregate([
  { $lookup: { from: "takes", localField: "_id", foreignField: "student_id", as: "courses" } },
  { $unwind: "$courses" },
  {
    $lookup: {
      from: "sections",
      let: { cid: "$courses.course_id", sid: "$courses.sec_id", sem: "$courses.semester", yr: "$courses.year" },
      pipeline: [
        { $match: {
            $expr: {
              $and: [
                { $eq: ["$course_id", "$$cid"] },
                { $eq: ["$sec_id", "$$sid"] },
                { $eq: ["$semester", "$$sem"] },
                { $eq: ["$year", "$$yr"] },
                { $eq: ["$building", "Watson"] }
              ]
            }
          }
        }
      ],
      as: "section"
    }
  },
  { $match: { section: { $ne: [] } } },
  { $group: { _id: "$_id", name: { $first: "$name" } } },
  { $project: { name: 1, _id: 0 } }
])
```

#### Neo4j (Graph Natural Fit!)
```cypher
MATCH (s:Student)-[:ENROLLED_IN]->(c:Course)-[:HELD_IN]->(b:Building {name: "Watson"})
RETURN DISTINCT s.name
```

Or with Section node:
```cypher
MATCH (s:Student)-[:TAKES]->(sec:Section)-[:IN_BUILDING]->(b:Building {name: "Watson"})
RETURN DISTINCT s.name
```

#### Cassandra (Multi-Query Approach)
```sql
-- Query 1: Find sections in Watson
SELECT course_id, sec_id, semester, year
FROM sections_by_building
WHERE building = 'Watson';

-- Query 2: For each section, find enrolled students (in app code)
SELECT student_id, student_name
FROM students_by_section
WHERE course_id = 'CS-101'
  AND sec_id = '1'
  AND semester = 'Fall'
  AND year = 2017;
```

#### Redis (Requires Multiple Lookups)
```bash
# 1. Get sections in Watson building
SMEMBERS building:Watson:sections
# Returns: "CS-101:1:Fall:2017", "BIO-301:1:Spring:2018"

# 2. For each section, get students
SMEMBERS section:CS-101:1:Fall:2017:students
# Returns student IDs

# 3. Get student names
HGET student:00128 name
```

#### XQuery
```xquery
for $student in doc("university.xml")//Student
where $student/Enrollments/Enrollment/Section/Building = "Watson"
return <StudentName>{$student/Name/text()}</StudentName>
```

---

## Decision Tree: Which Database When?

```
START: What is your primary use case?
│
├─ Need ACID transactions + complex queries?
│  └─> SQL (PostgreSQL, MySQL)
│
├─ Flexible schema + nested data?
│  └─> MongoDB
│
├─ Extreme write throughput + massive scale?
│  └─> Cassandra
│
├─ Ultra-fast reads + caching?
│  └─> Redis
│
├─ Relationship-heavy queries?
│  └─> Neo4j
│
└─ Hierarchical data + validation?
   └─> XML/XQuery
```

### Detailed Decision Matrix

| Use Case | Best Choice | Why | Avoid |
|----------|------------|-----|-------|
| Banking transactions | SQL | ACID, consistency critical | Cassandra, Redis |
| Social network graph | Neo4j | Relationship traversal | SQL (join cost), MongoDB |
| Real-time leaderboard | Redis | In-memory speed | Cassandra, XML |
| Product catalog | MongoDB | Flexible attributes | SQL (rigid schema) |
| IoT sensor data | Cassandra | High write throughput | SQL (scalability) |
| Document storage with schema | XML/JSON | Validation, hierarchy | Redis (no structure) |
| E-commerce cart (session) | Redis | TTL, fast access | Neo4j, XML |
| Recommendation engine | Neo4j | "who bought X also bought Y" | SQL, Cassandra |
| Analytics on structured data | SQL | Mature query tools | Redis |
| Chat message history | Cassandra | Time-series + scale | Neo4j, XML |

---

## Practical Translation Workflow

When translating a query across databases:

### Step 1: Understand the Query Intent
- What data do you need?
- What relationships matter?
- What's the access pattern?

### Step 2: Map to Data Model
- **SQL**: Which tables? What joins?
- **MongoDB**: Embedded or referenced?
- **Cassandra**: What's the partition key?
- **Redis**: What's the key pattern?
- **Neo4j**: What nodes/relationships?
- **XML**: What's the hierarchy?

### Step 3: Translate Operations
- **Filter** → WHERE / $match / WHERE / key pattern / WHERE / XPath predicate
- **Join** → JOIN / $lookup / denormalize / multi-get / MATCH / nested elements
- **Aggregate** → GROUP BY / $group / app code / sorted sets / COUNT / XQuery let
- **Sort** → ORDER BY / .sort() / clustering key / ZRANGE / ORDER BY / order by

### Step 4: Optimize for the Database
- **SQL**: Indexes, query plans
- **MongoDB**: Indexes, embed vs. reference
- **Cassandra**: Partition key design, denormalization
- **Redis**: Key naming, data structures
- **Neo4j**: Relationship indexing, path lengths
- **XML**: Index paths, minimize tree depth

---

## Summary Table: Query Pattern Support

| Query Type | SQL | MongoDB | Cassandra | Redis | Neo4j | XML |
|------------|-----|---------|-----------|-------|-------|-----|
| Filter by field | ✅ Easy | ✅ Easy | ✅ If partition key | ⚠️ Scan | ✅ Easy | ✅ Easy |
| Join 2 tables | ✅ Native | ⚠️ $lookup | ❌ App code | ❌ App code | ✅ Native | ⚠️ Nested |
| Join 3+ tables | ✅ Easy | ⚠️ Complex | ❌ App code | ❌ App code | ✅ Easy | ⚠️ Complex |
| Aggregation | ✅ GROUP BY | ✅ $group | ⚠️ Limited | ⚠️ App code | ✅ COUNT | ✅ let/sum |
| Sort | ✅ ORDER BY | ✅ .sort() | ⚠️ Clustering | ✅ Sorted Set | ✅ ORDER BY | ✅ order by |
| Pagination | ✅ LIMIT/OFFSET | ✅ skip/limit | ✅ Paging state | ⚠️ Manual | ✅ SKIP/LIMIT | ✅ subsequence |
| Transactions | ✅ ACID | ⚠️ Single-doc | ❌ Lightweight | ⚠️ MULTI/EXEC | ✅ ACID | ❌ |
| Range queries | ✅ Easy | ✅ $gte/$lte | ⚠️ Clustering col | ✅ ZRANGEBYSCORE | ⚠️ Indexed | ✅ Comparison |
| Text search | ✅ LIKE/FTS | ✅ $text | ❌ | ❌ | ✅ CONTAINS | ✅ contains() |
| Graph traversal | ⚠️ Recursive | ❌ | ❌ | ❌ | ✅ Native | ❌ |

**Legend**: ✅ = Well supported, ⚠️ = Possible but awkward, ❌ = Not supported

---

## Key Takeaways

1. **No Single "Best" Database**: Each paradigm excels at specific workloads
2. **Data Modeling is Different**: Don't force relational thinking into NoSQL
3. **Query Patterns Drive Design**: In Cassandra/Redis, design tables/keys for your queries
4. **Trade-offs are Real**: Speed vs. consistency, flexibility vs. structure
5. **Polyglot Persistence**: Real systems use multiple databases (SQL + Redis + MongoDB)

**Remember**: The best database is the one that matches your access patterns! 🚀
