# CS G516 — Advanced Database Systems
### Comprehensive Database Guide
*Syntax · Transitions · Advantages · Disadvantages · Real-World Use Cases*
*BITS Pilani, Rajasthan 333031*

---

> **📚 What This Guide Covers**
>
> This guide covers every database technology in CS G516: **MySQL** (Relational), **MongoDB** (Document), **Redis** (Key-Value), **Apache Cassandra** (Wide-Column), **Neo4j** (Graph), **XML/XQuery** (Hierarchical), and **ObjectDB** (Object-Oriented). Each section covers: core concepts, full syntax with examples, when to use it, advantages, and disadvantages.

---

## 📑 Table of Contents

1. [The Big Picture — All 7 Database Models](#1-the-big-picture--all-7-database-models)
2. [MySQL — Relational Database](#2-mysql--relational-database)
3. [MongoDB — Document Database](#3-mongodb--document-database)
4. [Redis — Key-Value Store](#4-redis--key-value-store)
5. [Apache Cassandra — Wide-Column Store](#5-apache-cassandra--wide-column-store)
6. [Neo4j — Graph Database](#6-neo4j--graph-database)
7. [XML, DTD, XPath & XQuery](#7-xml-dtd-xpath--xquery)
8. [ObjectDB — Object-Oriented Database](#8-objectdb--object-oriented-database)
9. [Transition Cheat Sheet](#9-transition-cheat-sheet)

---

## 1. The Big Picture — All 7 Database Models

Modern software uses many types of databases. Each was invented to solve problems that other models handle poorly.

| Database | Model | Data Unit | Query Language | Best For | CAP |
|---|---|---|---|---|---|
| MySQL | Relational | Row / Table | SQL | Structured, transactional data | CA |
| MongoDB | Document | BSON Document | MQL / Aggregation | Flexible JSON-like records | CP/AP |
| Redis | Key-Value | Key → Value | Redis Commands | Cache, sessions, leaderboards | CP/AP |
| Cassandra | Wide-Column | Row in partition | CQL | Time-series, massive write scale | AP |
| Neo4j | Graph | Node + Edge | Cypher | Relationships, social, routing | CA/CP |
| XML/BaseX | Hierarchical | XML Document | XPath / XQuery | Documents, configs, data exchange | N/A |
| ObjectDB | Object-Oriented | Java Object | JPQL | OOP apps, no impedance mismatch | CA |

### The CAP Theorem — Why No Database Does Everything

Every distributed database can only guarantee **2 of 3** properties simultaneously:

- **C — Consistency**: Every read returns the most recent write
- **A — Availability**: Every request gets a response (may not be latest)
- **P — Partition Tolerance**: System works even if network splits occur

> 💡 **Key Insight:** In practice, network partitions always happen. So you're really choosing between **CP** (consistent but may be unavailable during partition) and **AP** (always available but may serve stale data). MySQL is CA only in single-node setups.

---

## 2. MySQL — Relational Database

MySQL stores data in **tables** with rows and columns, enforces strict schemas, and supports ACID transactions. It's the gold standard for structured, transactional workloads.

### 2.1 Core Concepts

| Concept | Description | Example |
|---|---|---|
| Table | 2D structure with typed columns and rows | `CREATE TABLE students (id INT, name VARCHAR(100))` |
| Primary Key | Unique row identifier, never NULL | `id INT PRIMARY KEY AUTO_INCREMENT` |
| Foreign Key | References PK in another table | `FOREIGN KEY (dept_id) REFERENCES dept(id)` |
| Index | Speeds up lookups on a column | `CREATE INDEX idx_name ON students(name)` |
| JOIN | Combines rows from multiple tables | `SELECT * FROM a JOIN b ON a.id = b.a_id` |
| Transaction | Group of operations that succeed or fail together | `BEGIN; UPDATE ...; COMMIT;` |
| View | Saved SELECT as a virtual table | `CREATE VIEW v AS SELECT ...` |

---

### 2.2 DDL Syntax

#### CREATE TABLE — with all constraint types

```sql
CREATE TABLE orders (
    order_id    INT             PRIMARY KEY AUTO_INCREMENT,
    customer_id INT             NOT NULL,
    amount      DECIMAL(10,2)   NOT NULL CHECK (amount > 0),
    status      VARCHAR(20)     DEFAULT 'pending',
    email       VARCHAR(100)    UNIQUE,
    created_at  DATETIME        DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_customer
        FOREIGN KEY (customer_id) REFERENCES customers(id)
        ON DELETE CASCADE ON UPDATE SET NULL
);
```

#### ALTER TABLE — modifying an existing table

```sql
ALTER TABLE orders ADD COLUMN notes TEXT;
ALTER TABLE orders MODIFY COLUMN status VARCHAR(30) NOT NULL;
ALTER TABLE orders CHANGE COLUMN email email_addr VARCHAR(150);
ALTER TABLE orders DROP COLUMN notes;
ALTER TABLE orders RENAME TO purchase_orders;
```

---

### 2.3 DML Syntax

#### SELECT — querying data

```sql
-- Basic SELECT
SELECT name, salary FROM instructors WHERE dept = 'CS';

-- Aggregate functions
SELECT dept, COUNT(*) AS total, AVG(salary) AS avg_sal
FROM instructors
GROUP BY dept
HAVING AVG(salary) > 70000
ORDER BY avg_sal DESC;

-- JOINs across 3 tables
SELECT s.name, c.title
FROM students s
JOIN enrollments e ON s.id = e.student_id
JOIN courses c     ON e.course_id = c.id
WHERE c.credits = 3;

-- Subquery
SELECT name FROM instructors
WHERE salary = (SELECT MAX(salary) FROM instructors);
```

#### INSERT / UPDATE / DELETE

```sql
INSERT INTO students VALUES (1, 'Alice', 21, 'CS');
INSERT INTO students SELECT id+1000, name, age, dept FROM old_students;

UPDATE instructors SET salary = salary * 1.10 WHERE dept = 'CS';

DELETE FROM students WHERE age < 18;
```

---

### 2.4 Date Functions (Lab 2.1)

```sql
SELECT NOW(), CURRENT_DATE(), CURRENT_TIME();
SELECT YEAR('2024-06-15'), MONTH('2024-06-15'), DAY('2024-06-15');
SELECT DAYNAME('2024-06-15'), MONTHNAME('2024-06-15');
SELECT DATE_ADD('2024-01-01', INTERVAL 3 MONTH);        -- 2024-04-01
SELECT DATE_SUB('2024-06-15', INTERVAL 7 DAY);
SELECT DATEDIFF('2024-12-31', '2024-01-01');             -- 365
SELECT TIMESTAMPDIFF(MONTH, '2024-01-01', '2024-06-15'); -- 5
SELECT LAST_DAY('2024-02-01');                           -- 2024-02-29
```

#### Self-Join for consecutive date patterns

```sql
SELECT a.sailor_id, a.reserve_date, b.reserve_date AS next_day
FROM reserves a
JOIN reserves b ON a.sailor_id = b.sailor_id
               AND DATEDIFF(b.reserve_date, a.reserve_date) = 1;
```

---

### 2.5 UDFs, Triggers & Stored Procedures (Lab 2.2)

#### User Defined Function

```sql
DELIMITER $$
CREATE FUNCTION discounted_price(price DECIMAL(10,2), pct DECIMAL(5,2))
RETURNS DECIMAL(10,2) DETERMINISTIC
BEGIN
    RETURN price * (1 - pct/100);
END$$
DELIMITER ;

-- Usage:
SELECT meal_name, discounted_price(price, 15) FROM meals;
```

#### BEFORE INSERT Trigger — validation

```sql
DELIMITER $$
CREATE TRIGGER validate_price
BEFORE INSERT ON meals
FOR EACH ROW
BEGIN
    DECLARE avg_price DECIMAL(10,2);
    SELECT AVG(price) INTO avg_price FROM meals;
    IF NEW.price > avg_price * 3 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Price too high';
    END IF;
END$$
DELIMITER ;
```

#### AFTER UPDATE Trigger — audit log

```sql
DELIMITER $$
CREATE TRIGGER audit_salary
AFTER UPDATE ON instructors
FOR EACH ROW
BEGIN
    INSERT INTO salary_audit(instructor_id, old_salary, new_salary, changed_at)
    VALUES (OLD.id, OLD.salary, NEW.salary, NOW());
END$$
DELIMITER ;
```

#### Stored Procedure

```sql
DELIMITER $$
CREATE PROCEDURE top_meals(IN n INT)
BEGIN
    SELECT meal_name, price FROM meals ORDER BY price DESC LIMIT n;
END$$
DELIMITER ;

CALL top_meals(5);
```

---

### 2.6 MySQL — Advantages & Disadvantages

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| ACID transactions — data is always consistent | Schema changes on large tables are slow/expensive |
| SQL is universally known — easy to hire for | Vertical scaling only (sharding is complex) |
| Mature ecosystem: tools, backups, monitoring | Fixed schema — adding fields requires ALTER TABLE |
| Strong data integrity via constraints + FKs | JOINs on huge tables can be slow |
| Complex queries: subqueries, aggregates, window fns | Not ideal for unstructured or hierarchical data |
| Views, triggers, stored procedures built-in | Row locking under high write concurrency |

> 🎯 **Use MySQL When...** You need ACID transactions (banking, e-commerce orders). Your data is naturally tabular with well-defined relationships. Examples: user accounts, order management, financial records, HR systems.

---

## 3. MongoDB — Document Database

MongoDB stores data as **BSON (Binary JSON) documents** inside collections. There is no fixed schema — each document can have different fields.

### 3.1 How MongoDB Thinks Differently from MySQL

| MySQL Term | MongoDB Term | Key Difference |
|---|---|---|
| Database | Database | Same concept |
| Table | Collection | No fixed schema |
| Row | Document | JSON/BSON — can nest, can have arrays |
| Column | Field | Fields can differ per document |
| JOIN | `$lookup` | Done in aggregation pipeline — avoid if possible |
| Primary Key | `_id` field | Auto-generated ObjectId if not specified |
| GROUP BY | `$group` stage | Part of aggregation pipeline |

---

### 3.2 CRUD Operations

#### INSERT

```js
// Single document
db.drivers.insertOne({
  driverId: 1,
  name: "Lewis Hamilton",
  nationality: "British",
  championships: 7,
  teams: ['Mercedes', 'McLaren']   // embedded array
})

// Multiple documents
db.drivers.insertMany([
  { driverId: 2, name: "Max Verstappen", nationality: "Dutch" },
  { driverId: 3, name: "Fernando Alonso", nationality: "Spanish" }
])
```

#### SELECT (find)

```js
// All documents
db.drivers.find({})

// Filter: British drivers
db.drivers.find({ nationality: "British" })

// Range query
db.drivers.find({ championships: { $gt: 3 } })

// Multiple conditions (AND implied)
db.drivers.find({
  nationality: { $in: ["British", "German"] },
  championships: { $gte: 5 }
})

// Projection — include only specific fields, exclude _id
db.drivers.find({}, { name: 1, nationality: 1, _id: 0 })

// Sort + Limit
db.drivers.find({}).sort({ championships: -1 }).limit(5)
```

#### UPDATE

```js
// Update one document
db.drivers.updateOne(
  { name: "Lewis Hamilton" },
  { $set: { championships: 7 }, $inc: { wins: 1 } }
)

// Update many
db.drivers.updateMany(
  { nationality: "British" },
  { $set: { license_country: "UK" } }
)
```

#### DELETE

```js
db.drivers.deleteOne({ driverId: 1 })
db.drivers.deleteMany({ championships: { $lt: 1 } })
```

---

### 3.3 Aggregation Pipeline

The aggregation pipeline is MongoDB's most powerful feature. Data flows through **stages** like a Unix pipe.

```js
// Count drivers per nationality, sort by most common
db.drivers.aggregate([
  { $match: { championships: { $gt: 0 } } },   // filter first
  { $group: {
      _id: "$nationality",
      count: { $sum: 1 },
      total_wins: { $sum: "$wins" }
  }},
  { $sort: { count: -1 } },
  { $limit: 10 }
])
```

#### $lookup — Joining two collections

```js
db.results.aggregate([
  { $match: { year: 2021, position: 1 } },
  { $lookup: {
      from: "drivers",          // other collection
      localField: "driverId",   // field in results
      foreignField: "driverId", // matching field in drivers
      as: "driver_info"
  }},
  { $unwind: "$driver_info" },  // flatten the joined array
  { $project: {
      race: 1,
      winner: "$driver_info.name",
      _id: 0
  }}
])
```

#### $unwind — Decomposing arrays

```js
// Each team entry becomes its own document
db.drivers.aggregate([
  { $unwind: "$teams" },
  { $group: { _id: "$teams", driver_count: { $sum: 1 } } }
])
```

---

### 3.4 MongoDB — Advantages & Disadvantages

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Flexible schema — add fields without migration | No native multi-document ACID (pre-4.0) |
| Documents map naturally to OOP objects | `$lookup` (joins) are expensive — design to avoid them |
| Horizontal scaling via sharding (built-in) | Data duplication expected — no normalization |
| Rich query language + powerful aggregation | No CHECK constraints or FK enforcement |
| Handles nested/array data naturally | Large documents increase memory usage |
| Great developer experience, easy prototyping | Schema validation must be done at app layer |

> 🎯 **Use MongoDB When...** Your schema changes frequently (startups, agile projects). Data is naturally document-shaped. Examples: content management systems, e-commerce catalogs, user profiles, mobile app backends.

---

## 4. Redis — Key-Value Store

Redis is an **in-memory data structure store** — blazing fast (sub-millisecond). It supports rich data types and is commonly used for caching, sessions, leaderboards, and pub/sub messaging.

### 4.1 Strings — the simplest type

```redis
SET user:1001:name 'Alice'
GET user:1001:name              -- 'Alice'
SETEX session:abc123 3600 'user_data'  -- expires in 3600s
INCR page:views                 -- atomic counter ++
INCRBY product:42:stock -5      -- decrement by 5
TTL session:abc123              -- seconds remaining
PERSIST session:abc123          -- remove expiry
```

### 4.2 Hashes — like a row in a table

```redis
HSET player:10 name 'Mbappe' team 'PSG' goals 30 age 25
HGET player:10 name             -- 'Mbappe'
HGETALL player:10               -- all field-value pairs
HMGET player:10 name goals      -- multiple fields at once
HINCRBY player:10 goals 5       -- increment a field
HDEL player:10 age              -- remove a field
```

### 4.3 Lists — ordered, duplicates allowed

```redis
LPUSH match:queue match:101 match:102   -- push to front
RPUSH match:queue match:103             -- push to back
LRANGE match:queue 0 -1                -- get all elements
LPOP match:queue                       -- remove from front
RPOP match:queue                       -- remove from back
LTRIM match:queue 0 9                  -- keep only first 10
```

### 4.4 Sets — unordered, unique values

```redis
SADD league:premier team:1 team:2 team:3
SADD league:la_liga team:4 team:2 team:5
SMEMBERS league:premier           -- all members
SISMEMBER league:premier team:1   -- check membership -> 1 or 0
SINTER league:premier league:la_liga  -- teams in BOTH leagues
SUNION league:premier league:la_liga  -- all teams combined
SDIFF  league:premier league:la_liga  -- only in premier
```

### 4.5 Sorted Sets (ZSets) — ranked with scores

```redis
ZADD leaderboard 2500 'player:alice'
ZADD leaderboard 3100 'player:bob'
ZADD leaderboard 2800 'player:carol'
ZREVRANGE leaderboard 0 2 WITHSCORES   -- top 3 with scores
ZRANK leaderboard 'player:alice'        -- rank (0-indexed from lowest)
ZINCRBY leaderboard 200 'player:alice'  -- add to score
ZCOUNT leaderboard 2000 3000            -- count in score range
ZRANGEBYSCORE leaderboard 2500 3000     -- members in score range
```

### 4.6 TTL & Transactions

```redis
EXPIRE user:session:xyz 1800    -- expire in 30 minutes

-- Atomic transaction block
MULTI
  HSET match:live:202 home_score 2
  HINCRBY match:live:202 home_score 1
EXEC
```

### 4.7 SCAN — safe key iteration

```redis
SCAN 0 MATCH player:* COUNT 100   -- returns cursor + keys
-- Keep calling with returned cursor until cursor = 0
-- NEVER use KEYS * in production — it blocks the server
```

### 4.8 Pub/Sub — messaging

```redis
-- Terminal 1 (subscriber):
SUBSCRIBE match:updates

-- Terminal 2 (publisher):
PUBLISH match:updates 'goal:team_a:75min'

-- Pattern subscribe (all match channels):
PSUBSCRIBE match:*
```

> 📛 **Key Naming Convention:** Use hierarchical names with colons: `object:id:field`
> Examples: `user:1001:profile` | `match:live:42` | `leaderboard:season:2024`
> This makes SCAN patterns easy and keys human-readable.

---

### 4.9 Redis — Advantages & Disadvantages

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Sub-millisecond latency — fastest database available | Data size limited by RAM — expensive to scale |
| Rich data structures (not just key-value) | No complex queries — no JOINs, no GROUP BY |
| Built-in TTL / expiry for session management | Persistence is optional (default is in-memory only) |
| Pub/Sub messaging system built-in | No schema enforcement — easy to store wrong data |
| Atomic operations (INCR, MULTI/EXEC) | Not suitable as primary database for large datasets |
| Perfect as cache layer in front of any DB | Single-threaded command processing |

> 🎯 **Use Redis When...** You need ultra-fast reads/writes (caching API responses). Session storage with automatic expiry. Real-time leaderboards. Examples: session store, API cache, live scoreboards, job queues.

---

## 5. Apache Cassandra — Wide-Column Store

Cassandra is designed for **massive-scale write-heavy workloads** with no single point of failure. It uses a masterless ring architecture. Data is modelled around queries — the schema is designed to serve specific read patterns.

### 5.1 Cassandra vs MySQL — The Mental Model Shift

| MySQL Thinking | Cassandra Thinking |
|---|---|
| Design tables to model data (normalization) | Design tables to answer specific queries (query-first) |
| One table per entity, JOIN as needed | Duplicate data across tables — JOINs don't exist |
| Add WHERE on any column (with index) | WHERE must use the partition key first |
| Rows are uniform (same columns) | Rows can have different columns in same partition |
| DELETE creates immediate space | DELETE creates a tombstone, cleaned up later |
| Single server, vertical scaling | Horizontal ring — add nodes for more capacity |

---

### 5.2 Keyspace & Table Creation

```sql
-- Create keyspace (equivalent to a database)
CREATE KEYSPACE football
WITH replication = {
  'class': 'SimpleStrategy',
  'replication_factor': 3    -- 3 copies across nodes
};
USE football;

-- Table designed for "get matches by date" query
CREATE TABLE matches_by_date (
  match_date   DATE,    -- partition key (determines which node)
  match_id     UUID,    -- clustering key (sort order within partition)
  home_team    TEXT,
  away_team    TEXT,
  home_score   INT,
  away_score   INT,
  PRIMARY KEY (match_date, match_id)
);

-- Separate table for "get matches by team" query
CREATE TABLE matches_by_team (
  team_name    TEXT,    -- partition key
  match_date   DATE,    -- clustering key
  match_id     UUID,
  opponent     TEXT,
  result       TEXT,
  PRIMARY KEY (team_name, match_date)
) WITH CLUSTERING ORDER BY (match_date DESC);
```

---

### 5.3 CQL CRUD Operations

#### INSERT

```sql
INSERT INTO matches_by_date (match_date, match_id, home_team, away_team)
VALUES ('2024-03-15', uuid(), 'Arsenal', 'Chelsea');

-- With TTL (data expires after 90 days)
INSERT INTO live_scores (match_id, score)
VALUES (uuid(), '2-1') USING TTL 7776000;
```

#### SELECT — partition key is REQUIRED

```sql
-- GOOD: uses partition key
SELECT * FROM matches_by_date WHERE match_date = '2024-03-15';

-- GOOD: partition key + clustering key range
SELECT * FROM matches_by_date
WHERE match_date = '2024-03-15'
AND match_id > some_uuid;

-- BAD — triggers ALLOW FILTERING (dangerous at scale!)
SELECT * FROM matches_by_date WHERE home_team = 'Arsenal';
```

#### UPDATE and DELETE

```sql
UPDATE matches_by_date SET home_score = 3
WHERE match_date = '2024-03-15' AND match_id = some_uuid;

DELETE FROM matches_by_date
WHERE match_date = '2024-03-15' AND match_id = some_uuid;
```

#### Collection types (List, Set, Map)

```sql
ALTER TABLE players ADD past_teams LIST<TEXT>;
ALTER TABLE players ADD awards    SET<TEXT>;
ALTER TABLE players ADD stats     MAP<TEXT, INT>;

UPDATE players SET
  past_teams = past_teams + ['Barcelona'],
  awards     = awards + {'Ballon d Or'},
  stats['goals'] = 45
WHERE player_id = some_uuid;
```

#### Materialized View

```sql
CREATE MATERIALIZED VIEW players_by_team AS
  SELECT * FROM players
  WHERE team IS NOT NULL AND player_id IS NOT NULL
  PRIMARY KEY (team, player_id);
-- Auto-synced whenever the players table is updated
```

---

### 5.4 Cassandra — Advantages & Disadvantages

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Linearly scalable — add nodes for more throughput | No JOINs — denormalization required |
| No single point of failure (masterless) | No ACID transactions across partitions |
| Handles millions of writes/second | ALLOW FILTERING is dangerous at scale |
| Tunable consistency (ONE, QUORUM, ALL) | Schema design is hard — must know queries upfront |
| Built-in TTL for time-series data | Tombstones accumulate — deletes have overhead |
| Multi-datacenter replication built-in | Aggregations are limited within a single partition |

> 🎯 **Use Cassandra When...** You need to write data at massive scale (IoT sensors, logs, metrics). Reads are always by a known partition key. Examples: Netflix viewing history, IoT telemetry, time-series sensor data, activity feeds.

---

## 6. Neo4j — Graph Database

Neo4j stores data as **nodes** (entities) connected by **relationships** (edges). Both nodes and relationships can have properties. This makes traversing complex relationships trivially easy — no JOINs needed.

### 6.1 Graph Model Components

| Component | Description | Example |
|---|---|---|
| Node | An entity (like a row) | `(:Person {name: 'Alice'})` |
| Label | Type tag on a node (like a table name) | `:Person`, `:City`, `:Product` |
| Relationship | Directed edge between nodes | `-[:KNOWS]->` |
| Properties | Key-value pairs on nodes or relationships | `{since: 2020, weight: 5}` |
| Pattern | ASCII-art describing a graph shape | `(a)-[:LINKED]->(b)` |

---

### 6.2 Cypher — CREATE

```cypher
// Create a standalone node
CREATE (:DistributionCenter { name: 'Madrid', country: 'Spain', planningValue: 95 })

// Create a node and capture it in a variable
CREATE (dc:DistributionCenter { name: 'Tokyo', country: 'Japan' })
RETURN dc

// Create two nodes with a relationship in one query
CREATE (madrid:DC { name: 'Madrid' })
  -[:SUPPLY_TO { distance_km: 1200 }]->
  (paris:DC { name: 'Paris' })

// Create relationship between existing nodes
MATCH (a:DC { name: 'Madrid' }), (b:DC { name: 'Paris' })
CREATE (a)-[:SUPPLY_TO { distance_km: 1200 }]->(b)
```

---

### 6.3 Cypher — MATCH (SELECT)

```cypher
// Get all DistributionCenters in Spain
MATCH (dc:DistributionCenter)
WHERE dc.country = 'Spain'
RETURN dc.name, dc.planningValue
ORDER BY dc.planningValue DESC

// Multi-value filter
MATCH (dc:DistributionCenter)
WHERE dc.country IN ['Spain', 'Canada', 'Japan']
RETURN dc

// Traverse a relationship
MATCH (source:DC { name: 'Abu Dhabi' })-[:SUPPLY_TO]->(dest:DC)
RETURN dest.name, dest.country

// Count nodes per country (implicit GROUP BY)
MATCH (dc:DistributionCenter)
RETURN dc.country, COUNT(dc) AS total
ORDER BY total DESC

// Multi-hop path: Truck -> Route -> MagicPlace
MATCH (t:Truck { name: 'Truck3' })-[:ON_ROUTE]->(r:Route)-[:HAS_PROBLEM]->(p)
RETURN t.name, r.name, p

// Variable-length path (1 to 3 hops)
MATCH (start:DC { name: 'Madrid' })-[:LINKED*1..3]->(dest:DC)
WHERE dest.name <> 'Madrid'
RETURN DISTINCT dest.name

// Shortest path between two cities
MATCH p = shortestPath(
  (a:DC { name: 'Madrid' })-[:LINKED*]-(b:DC { name: 'Beijing' })
)
RETURN p, length(p) AS hops
```

---

### 6.4 Cypher — UPDATE & DELETE

```cypher
// Add / update a property
MATCH (dc:DC { name: 'Madrid' })
SET dc.planningValue = 98, dc.updated = true
RETURN dc

// Remove a property
MATCH (dc:DC { name: 'Madrid' })
REMOVE dc.outdated_field

// Delete a node (must have no relationships)
MATCH (dc:DC { name: 'OldHub' })
DELETE dc

// Delete a node AND all its relationships
MATCH (dc:DC { name: 'OldHub' })
DETACH DELETE dc
```

---

### 6.5 Neo4j — Advantages & Disadvantages

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Relationship traversal is O(1) — no JOIN overhead | Not suited for tabular/aggregate-heavy workloads |
| `shortestPath()` and graph algorithms built-in | Smaller ecosystem than MySQL or MongoDB |
| Cypher is intuitive — ASCII-art patterns | Less efficient for full-table scans vs relational |
| Perfect for recursive / deep relationships | Horizontal scaling is complex (sharding graphs is hard) |
| Properties on relationships (unique feature!) | Learning curve for thinking in graph patterns |
| ACID transactions on graph operations | Poor at bulk aggregations across all nodes |

> 🎯 **Use Neo4j When...** Your data is fundamentally about relationships (social networks, org charts, routing). You need to find paths, detect communities, or traverse many hops. Examples: fraud detection, recommendation engines, supply chain routing, knowledge graphs.

---

## 7. XML, DTD, XPath & XQuery

XML (eXtensible Markup Language) is a hierarchical, self-describing data format widely used in document exchange, configuration, and enterprise integration. **BaseX** is a native XML database that supports XPath and XQuery.

### 7.1 XML Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE property_listings SYSTEM "listings.dtd">
<property_listings>
  <property id="P001">
    <address>123 Baker Street</address>
    <type>apartment</type>
    <price currency="GBP">450000</price>
    <bedrooms>3</bedrooms>
    <features>
      <feature>parking</feature>
      <feature>garden</feature>
    </features>
  </property>
</property_listings>
```

---

### 7.2 DTD — Document Type Definition

```xml
<!DOCTYPE property_listings [
  <!ELEMENT property_listings (property+)>
  <!ELEMENT property (address, type, price, bedrooms, features)>
  <!ATTLIST property id ID #REQUIRED>
  <!ELEMENT address (#PCDATA)>
  <!ELEMENT type    (#PCDATA)>
  <!ELEMENT price   (#PCDATA)>
  <!ATTLIST price currency CDATA #IMPLIED>
  <!ELEMENT bedrooms (#PCDATA)>
  <!ELEMENT features (feature*)>
  <!ELEMENT feature  (#PCDATA)>
]>
```

#### DTD Symbol Reference

| Symbol | Meaning | Example |
|---|---|---|
| `+` | One or more | `<!ELEMENT listings (property+)>` |
| `*` | Zero or more | `<!ELEMENT features (feature*)>` |
| `?` | Zero or one (optional) | `<!ELEMENT notes (#PCDATA)?>` |
| `(a, b)` | Sequence — a then b | `<!ELEMENT property (address, type)>` |
| `(a \| b)` | Choice — a or b | `<!ELEMENT status (sale \| rent)>` |
| `#PCDATA` | Parsed character data (text content) | `<!ELEMENT address (#PCDATA)>` |
| `#REQUIRED` | Attribute is mandatory | `<!ATTLIST p id ID #REQUIRED>` |

---

### 7.3 XPath Queries (in BaseX)

#### Basic path navigation

```xpath
doc('listings.xml')/property_listings/property     (: all properties :)
doc('listings.xml')//price                          (: any price anywhere :)
doc('listings.xml')/property_listings/property/@id  (: all id attributes :)
```

#### Predicates — filtering

```xpath
//property[bedrooms > 2]                    (: more than 2 bedrooms :)
//property[type = 'apartment']              (: apartments only :)
//property[price > 400000 and bedrooms = 3] (: compound filter :)
//property[last()]                          (: last property :)
//feature[. = 'parking']                   (: features with value 'parking' :)
```

#### XPath Functions

```xpath
count(//property)                           (: total count :)
max(//property/price)                       (: maximum price :)
min(//property/bedrooms)                    (: minimum bedrooms :)
//property[count(features/feature) > 2]    (: properties with more than 2 features :)
```

#### XPath Axes

```xpath
//price/parent::*                           (: parent of any price element :)
//property[1]/following-sibling::property   (: all properties after first :)
//property/descendant::*                    (: all descendants :)
```

---

### 7.4 XQuery — FLWOR Expressions

**FLWOR** = **F**or · **L**et · **W**here · **O**rder by · **R**eturn — XQuery's equivalent of SQL SELECT.

#### Basic FLWOR

```xquery
for $p in doc('listings.xml')//property
where $p/bedrooms > 2
order by $p/price descending
return $p/address
```

#### Constructing new XML output

```xquery
for $p in doc('listings.xml')//property
where $p/type = 'apartment'
return
  <r>
    <addr>{ $p/address/text() }</addr>
    <cost>{ $p/price/text() }</cost>
  </r>
```

#### Using let and if/then/else

```xquery
for $p in doc('listings.xml')//property
let $label := if ($p/price > 500000) then 'luxury' else 'standard'
return <item category='{ $label }'>{ $p/address/text() }</item>
```

#### Grouping with distinct-values

```xquery
for $type in distinct-values(doc('listings.xml')//type)
let $props := doc('listings.xml')//property[type = $type]
return
  <group type='{ $type }' count='{ count($props) }'>
    { for $p in $props return $p/address }
  </group>
```

---

### 7.5 MySQL — XML & JSON Import/Export

```sql
-- Import XML into MySQL
LOAD XML INFILE '/path/to/listings.xml'
INTO TABLE property_listings
ROWS IDENTIFIED BY '<property>'
(@id, @address, @type, @price, @bedrooms)
SET prop_id = @id, address = @address;

-- Export as JSON array
SELECT JSON_ARRAYAGG(
  JSON_OBJECT(
    'id',      prop_id,
    'address', address,
    'price',   price
  )
) FROM property_listings
INTO OUTFILE '/tmp/listings.json';

-- Import JSON into MySQL using JSON_TABLE
SELECT jt.*
FROM JSON_TABLE('[{"id":1,"price":450000}]', '$[*]'
  COLUMNS (
    id    INT     PATH '$.id',
    price DECIMAL PATH '$.price'
  )
) AS jt;
```

---

### 7.6 XML — Advantages & Disadvantages

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Self-describing — human readable with tags | Verbose — much more text than JSON for same data |
| DTD/XSD schema validation built-in | Parsing is slow compared to JSON or binary formats |
| XPath/XQuery are powerful for tree navigation | Not suitable as a general-purpose database |
| Universal support across all languages/systems | No native support for arrays (use repeated elements) |
| Attributes and namespaces provide rich metadata | Deep nesting becomes hard to read/maintain |

> 🎯 **Use XML When...** Exchanging data with enterprise systems (SOAP, EDI). Document-centric data (legal documents, books). Configuration files (Maven, Spring, Android layouts). When schema validation is required.

---

## 8. ObjectDB — Object-Oriented Database

ObjectDB is a pure Java object database. Instead of mapping objects to tables, you **store Java objects directly**. JPQL (Java Persistence Query Language) closely resembles SQL but operates on entity classes.

### 8.1 The Impedance Mismatch Problem

> 🔧 **What is Impedance Mismatch?**
> In relational databases, you store data in flat tables. But in OOP, you have objects with inheritance, nested objects, and collections. Converting between them (ORM mapping) requires extra code and loses fidelity. ObjectDB eliminates this by storing objects directly.

| Relational (MySQL) | Object-Oriented (ObjectDB) |
|---|---|
| Table: `students` | `@Entity class Student` |
| Column: `name VARCHAR(100)` | Field: `String name` |
| Column: `age INT` | Field: `int age` |
| Separate enrollment table | `List<String> subjects` (embedded) |
| JOIN to get courses | Direct navigation: `student.subjects` |
| `SELECT * FROM students` | `SELECT s FROM Student s` |

---

### 8.2 Entity Class Structure

```java
import javax.persistence.*;
import java.util.List;

@Entity
public class Student {
    @Id
    private int studentId;
    private String name;
    private int age;
    private String department;
    private List<String> subjects;   // embedded collection — no JOIN needed
    // getters, setters, constructors...
}
```

---

### 8.3 JPQL Full Syntax

#### SELECT — basic queries

```jpql
-- All students (alias is MANDATORY in JPQL)
SELECT s FROM Student s

-- Projection (specific fields only)
SELECT s.name, s.age FROM Student s

-- WHERE clause
SELECT s FROM Student s WHERE s.age > 20
SELECT s FROM Student s WHERE s.age BETWEEN 18 AND 25
SELECT s FROM Student s WHERE s.department = 'CS'

-- AND / OR
SELECT s FROM Student s
WHERE s.department = 'CS' AND s.age >= 20

-- ORDER BY
SELECT s FROM Student s ORDER BY s.age DESC
```

#### JOIN on collection fields

```jpql
-- Students enrolled in 'Databases' subject
SELECT s FROM Student s
JOIN s.subjects sub
WHERE sub = 'Databases'
```

#### Aggregation

```jpql
SELECT AVG(s.age)             FROM Student s
SELECT MAX(s.age), MIN(s.age) FROM Student s
SELECT COUNT(s)               FROM Student s WHERE s.department = 'CS'
```

#### GROUP BY

```jpql
-- Group by department
SELECT s.department, COUNT(s), AVG(s.age)
FROM Student s
GROUP BY s.department

-- Group by collection element
SELECT sub, COUNT(s)
FROM Student s JOIN s.subjects sub
GROUP BY sub
```

#### UPDATE and DELETE

```jpql
-- Update
UPDATE Student s SET s.age = s.age + 1
WHERE s.department = 'CS'

-- Delete
DELETE FROM Student s WHERE s.age < 18
```

#### INSERT (in ObjectDB Explorer)

```jpql
INSERT INTO Student(studentId, name, age, department)
VALUES (101, 'Alice', 22, 'CS')
```

---

### 8.4 ObjectDB — Advantages & Disadvantages

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| No impedance mismatch — store objects directly | Less known — smaller community and ecosystem |
| Collections embedded in object (no JOIN needed) | Not suitable for non-Java applications |
| JPA standard — portable to other JPA providers | Complex reporting queries are awkward vs SQL |
| Pure Java — embedded mode needs no server | No built-in horizontal scaling |
| Objects preserve inheritance hierarchy | ObjectDB Explorer is limited vs MySQL Workbench |
| JPQL is familiar if you know SQL | Real-world adoption is low vs relational/document DBs |

> 🎯 **Use ObjectDB When...** Building a pure Java application where ORM overhead is significant. Domain model is complex with deep inheritance. Examples: scientific simulation data, CAD/CAM data, game state management.

---

## 9. Transition Cheat Sheet

> This is the most important section for **exam preparation**. It shows how the same concept is expressed in each database system.

---

### 9.1 Fundamental Operations Compared

| Operation | MySQL | MongoDB | Redis | Cassandra | Neo4j | JPQL |
|---|---|---|---|---|---|---|
| **Insert** | `INSERT INTO t VALUES(...)` | `db.col.insertOne({})` | `SET key value` / `HSET` | `INSERT INTO t VALUES(...)` | `CREATE (:Label {props})` | `INSERT INTO Entity VALUES(...)` |
| **Read** | `SELECT * FROM t WHERE ...` | `db.col.find({filter})` | `GET key` / `HGETALL` | `SELECT * FROM t WHERE part_key=...` | `MATCH (n:Label) WHERE ... RETURN n` | `SELECT e FROM Entity e WHERE ...` |
| **Update** | `UPDATE t SET col=val WHERE ...` | `db.col.updateOne({filter},{$set:{}})` | `SET key newval` / `HSET` | `UPDATE t SET col=val WHERE ...` | `MATCH (n) SET n.prop=val` | `UPDATE Entity e SET e.field=val WHERE ...` |
| **Delete** | `DELETE FROM t WHERE ...` | `db.col.deleteOne({filter})` | `DEL key` | `DELETE FROM t WHERE part_key=...` | `MATCH (n) DETACH DELETE n` | `DELETE FROM Entity e WHERE ...` |
| **Count** | `SELECT COUNT(*) FROM t` | `db.col.countDocuments({})` | `LLEN list` / `SCARD set` | `SELECT COUNT(*) FROM t` (per partition) | `MATCH (n:L) RETURN COUNT(n)` | `SELECT COUNT(e) FROM Entity e` |
| **Sort** | `ORDER BY col DESC` | `.sort({field: -1})` | `ZREVRANGE` (sorted sets) | `CLUSTERING ORDER BY` | `ORDER BY n.prop DESC` | `ORDER BY e.field DESC` |

---

### 9.2 Aggregation / Grouping Compared

| Goal | MySQL | MongoDB | Cassandra | Neo4j | JPQL |
|---|---|---|---|---|---|
| **Count per group** | `SELECT dept, COUNT(*) FROM t GROUP BY dept` | `$group: {_id: '$dept', n: {$sum: 1}}` | `COUNT(*)` within partition only | `MATCH (n) RETURN n.dept, COUNT(n)` | `SELECT e.dept, COUNT(e) FROM Emp e GROUP BY e.dept` |
| **Average value** | `SELECT AVG(salary) FROM t` | `$group: {avg: {$avg: '$salary'}}` | `SELECT AVG(salary) WHERE part=...` | `MATCH (n) RETURN AVG(n.salary)` | `SELECT AVG(e.salary) FROM Emp e` |
| **Max value** | `SELECT MAX(salary) FROM t` | `$group: {max: {$max: '$salary'}}` | `SELECT MAX(goals) WHERE part=...` | `MATCH (n) RETURN MAX(n.salary)` | `SELECT MAX(e.salary) FROM Emp e` |
| **Filter groups** | `HAVING COUNT(*) > 5` | `{$match: {count: {$gt: 5}}}` after `$group` | Not supported across partitions | `WITH ... WHERE count > 5` | `HAVING COUNT(e) > 5` |

---

### 9.3 Schema & Data Modelling Philosophy

| Database | Schema Philosophy | Normalization | Joins |
|---|---|---|---|
| MySQL | Fixed schema, defined upfront | Fully normalized (3NF/BCNF) | Required — data split across tables |
| MongoDB | Flexible schema, per-document | Denormalized — embed related data | Optional — prefer embedded docs |
| Redis | Schema-less — key naming is your schema | N/A — flat key-value pairs | None — application-side logic |
| Cassandra | Fixed per table, query-first design | Heavily denormalized by design | None — duplicate data across tables |
| Neo4j | Labels + properties, flexible | Relationships replace join tables | Built into traversal — it's the point |
| XML/XQuery | DTD/XSD optional validation | Hierarchical — parent-child nesting | XQuery joins with multiple `doc()` calls |
| ObjectDB | Java class defines schema | OOP-normalized (inheritance, embedding) | None — navigate object references |

---

### 9.4 Common Mistakes to Avoid

| Database | ❌ Common Mistake | ✅ Correct Approach |
|---|---|---|
| MySQL | `SELECT *` without WHERE on 50M rows | Always add WHERE; use EXPLAIN to check index usage |
| MySQL | UPDATE without WHERE clause | Always test SELECT first, then UPDATE with same WHERE |
| MongoDB | Designing documents that need frequent `$lookup` | Embed related data; redesign document structure |
| MongoDB | No indexes on queried fields | Create indexes on fields used in `find()` filters |
| Redis | Using `KEYS *` in production | Always use `SCAN` with COUNT and MATCH pattern |
| Redis | Storing entire dataset in Redis | Redis is a cache/accelerator — primary data lives elsewhere |
| Cassandra | WHERE on non-partition key without ALLOW FILTERING | Design a separate table with the needed partition key |
| Cassandra | Designing table before knowing queries | Always start with: *what queries will I run?* |
| Neo4j | Trying to do GROUP BY on all nodes | Use MATCH with aggregation on specific subgraphs |
| Neo4j | Not using DETACH DELETE | Nodes with relationships cannot be deleted — use DETACH DELETE |
| JPQL | Forgetting mandatory alias in FROM | `FROM Student s` — the alias `s` is always required |
| JPQL | Querying `List<>` field without JOIN | `JOIN s.subjects sub` — must JOIN collection to query it |

---

### 9.5 Choosing the Right Database — Decision Guide

> 🛒 **E-Commerce Platform Example — using ALL 7 databases together:**
>
> - **Orders, payments, inventory** → MySQL *(ACID transactions critical)*
> - **Product catalog, descriptions** → MongoDB *(flexible attributes per category)*
> - **User sessions, cart** → Redis *(fast, TTL-based expiry)*
> - **Order events / audit log** → Cassandra *(append-only, time-series)*
> - **Product recommendations** → Neo4j *(who-bought-what relationships)*
> - **Product specification XMLs** → XML/BaseX *(document exchange with suppliers)*

| Data Characteristic | Best Database | Why |
|---|---|---|
| Strong consistency, complex queries | MySQL | ACID + JOINs + GROUP BY |
| Variable attributes per record | MongoDB | Flexible document schema |
| Need it in < 1ms, fits in RAM | Redis | In-memory, O(1) access |
| 10M+ writes/sec, time-series | Cassandra | Masterless, append-optimized |
| Multi-hop relationship traversal | Neo4j | Index-free adjacency |
| Document exchange, hierarchical | XML | Self-describing, schema validation |
| Java OOP, complex object graph | ObjectDB | No impedance mismatch |

---

### 9.6 ACID vs BASE — The Core Trade-off

| Property | ACID (MySQL, Neo4j, ObjectDB) | BASE (MongoDB, Cassandra, Redis) |
|---|---|---|
| **A = Atomicity** | All or nothing in a transaction | Operations may partially succeed |
| **C = Consistency** | DB is always in a valid state | Eventual consistency — will converge |
| **I = Isolation** | Transactions don't interfere | Weaker isolation for performance |
| **D = Durability** | Committed data survives crashes | Varies — Redis can lose data by default |
| **Trade-off** | Correctness over performance | Performance and availability over correctness |
| **Use case** | Banking, medical records, orders | Social media, IoT telemetry, caching |

---

*CS G516 — Advanced Database Systems · BITS Pilani, Rajasthan 333031*

*MySQL · MongoDB · Redis · Cassandra · Neo4j · XML · ObjectDB*
