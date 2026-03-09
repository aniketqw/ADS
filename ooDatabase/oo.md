# CS G516 – Advanced Database Systems
## Object-Oriented Database Concepts: Complete Elaboration

**Institution:** BITS-Pilani | **Instructor:** Dr. Yashvardhan Sharma

---

> **How to Use This Document**
> All OO database-related concepts from the Master Index are grouped into logical sections, each with a Technical Definition, Core Principles, Exam Context, and Practical Example. Concept numbers from the Master Index are shown in **[brackets]**.

---

## Table of Contents

1. [Foundations & Historical Context](#1-foundations--historical-context)
2. [Object-Oriented Database Systems (OODBMS)](#2-object-oriented-database-systems-oodbms)
3. [ODMG Standards — ODM, ODL, OQL](#3-odmg-standards--odm-odl-oql)
4. [Object Identity (OID)](#4-object-identity-oid)
5. [Complex Objects & Composition](#5-complex-objects--composition)
6. [Classes, Interfaces & Inheritance](#6-classes-interfaces--inheritance)
7. [Encapsulation & Methods](#7-encapsulation--methods)
8. [Object References & Path Expressions](#8-object-references--path-expressions)
9. [Object Persistence](#9-object-persistence)
10. [Persistent Programming Languages (C++ & Java)](#10-persistent-programming-languages-c--java)
11. [Object Query Language (OQL)](#11-object-query-language-oql)
12. [OQL vs SQL — Key Differences](#12-oql-vs-sql--key-differences)
13. [Extents, Keys & Factory Objects](#13-extents-keys--factory-objects)
14. [ODL — Object Definition Language](#14-odl--object-definition-language)
15. [Conceptual Design — EER to ODB Mapping](#15-conceptual-design--eer-to-odb-mapping)
16. [Object-Relational Mapping (ORM)](#16-object-relational-mapping-orm)
17. [System Comparisons](#17-system-comparisons)

---

## 1. Foundations & Historical Context

### **[1] Relational DBMS Limitations**

**Technical Definition:**
Relational DBMSs support only a small, fixed set of atomic data types (integers, dates, strings, decimals). This is fundamentally insufficient for applications that deal with complex, structured, or multimedia data.

**Core Principles:**
- Atomic data types cannot represent nested structures natively.
- No built-in support for user-defined behaviors or operations on data.
- Complex data must be manually decomposed into flat tables, losing semantic richness.
- No concept of object identity — two tuples with the same values are indistinguishable.

**Exam Context:**
- A common trap: students think normalization *solves* this limitation. Normalization addresses redundancy, not the inability to represent complex types.
- Exam questions often ask *why* RDBMS failed for CAD/CAM or multimedia — the answer is always about the mismatch between rich, nested data and flat table structures.

**Practical Example:**
Storing a CAD drawing with nested sub-components, geometric transformations, and behavioral simulations cannot be cleanly done in a flat relational table. Each sub-component would require many joins to reconstruct.

---

### **[2] Complex Data Applications**

**Technical Definition:**
Application domains that require data types and relationships beyond what traditional RDBMS can handle natively.

**Core Principles:**
Three canonical examples appear repeatedly in exams:

| Domain | Why Complex |
|--------|-------------|
| **CAD/CAM** | Components are hierarchically nested; operations (rotate, scale) are behaviors on data |
| **Multimedia Repositories** | Images, audio, video — large binary objects with metadata |
| **Document Management** | Documents contain nested sections, embedded objects, version histories |

**Exam Context:**
You may be asked to *justify* the need for an OODB by picking one of these domains. Always cite: nested structure, behavioral methods, and the inadequacy of pure relational joins.

---

### **[3] Object-Database Systems**

**Technical Definition:**
Database systems that extend traditional DBMS functionality with object-oriented concepts: identity, encapsulation, inheritance, and complex types.

**Core Principles:**
Two development paths emerged (a frequently tested distinction):

```
Object-Database Systems
         │
         ├── Path 1: OODBMS
         │     └── Add DBMS features TO a programming language
         │
         └── Path 2: ORDBMS
               └── Add OO features TO a relational database
```

**Practical Example:**
- Path 1 → ObjectStore (C++ with persistence)
- Path 2 → Oracle 8i (SQL extended with CREATE TYPE)

---

### **[10] OODBMS Historical Failure**

**Technical Definition:**
Object-oriented database systems, despite theoretical advantages, largely failed to displace RDBMS in commercial markets during the 1990s.

**Core Principles:**
Reasons for failure:
1. **No query optimization** comparable to RDBMS query planners.
2. **No declarative query language** — navigation was procedural (pointer chasing).
3. **Tight coupling** to a single programming language (usually C++).
4. **Poor support** for ad-hoc queries, transactions, crash recovery.
5. **Lack of standardization** — each vendor had different APIs.

**Exam Context:**
The exam may ask: "Why did OODBMS fail?" The answer is *efficiency and lack of high-level query language*, not poor data modeling.

---

### **[11] Object-Relational Success**

**Technical Definition:**
Object-relational systems succeeded by incorporating object-oriented features (complex types, inheritance, methods) while retaining the relational model's query language, transactions, and optimizer.

**Core Principles:**
- The **relation remains the fundamental abstraction** — you still query with SQL.
- Complex types are layered on top, not replacing the relational foundation.
- SQL:1999 (SQL3) standardized these extensions.
- Major vendors (IBM DB2, Oracle, Informix) adopted this approach.

**Exam Context:**
"O-R systems retained the ________ as the fundamental abstraction." Answer: **relation**.

---

## 2. Object-Oriented Database Systems (OODBMS)

### **[4] OODBMS Overview**

**Technical Definition:**
OODBMS can be understood as an attempt to add DBMS functionality (persistence, transactions, query, recovery) to an object-oriented programming language environment.

**Core Principles:**
Key DBMS features added to OO languages:
1. **Persistence** — objects survive beyond program execution.
2. **Transactions** — ACID properties for object operations.
3. **Query** — ability to search for objects by property values.
4. **Concurrency control** — multi-user access to shared objects.
5. **Recovery** — crash recovery for persistent objects.

**Exam Context:**
The distinction is direction of extension:
- OODBMS: Programming language → add DB features
- ORDBMS: Relational DB → add OO features

---

### **[146] Object Database Core Features**

**Technical Definition:**
The six defining characteristics that distinguish an OODBMS from a relational system.

**Core Principles:**
All six must be understood independently and together:

```
┌─────────────────────────────────────────────┐
│          OODBMS Core Features                │
│                                             │
│  1. Object Identity (OID)                   │
│  2. Complex Objects (Composition)           │
│  3. Classes and Inheritance                 │
│  4. Object References                       │
│  5. Persistence                             │
│  6. Encapsulation of Operations             │
└─────────────────────────────────────────────┘
```

Each is detailed in its own section below.

---

## 3. ODMG Standards — ODM, ODL, OQL

### **[5] ODMG (Object Database Management Group)**

**Technical Definition:**
An industry consortium that developed a set of standards for object-oriented databases, analogous to what ANSI/ISO did for SQL and relational databases.

**Core Principles:**
- Founded to address the lack of standards that contributed to OODBMS fragmentation.
- Produced three key standards: ODM, ODL, and OQL.
- Standards designed to be programming-language independent.

---

### **[6] Object Data Model (ODM)**

**Technical Definition:**
The conceptual data model underpinning ODL and OQL — defines the primitives: objects, literals, interfaces, classes, inheritance, and collections.

**Core Principles:**

**Objects vs. Literals:**

| | Object | Literal |
|---|---|---|
| Has OID? | Yes | No |
| Can be shared? | Yes (by reference) | No (copied by value) |
| Examples | A `Person` instance | An integer, a string |

**The Five Aspects of an Object [162]:**

| Aspect | Meaning |
|--------|---------|
| **Identifier** | Unique OID |
| **Name** | Optional human-readable entry point |
| **Lifetime** | Transient or persistent |
| **Structure** | Type definition (attributes, relationships) |
| **Creation** | How the object is instantiated |

---

### **[153] ODMG Standards Components**

**Technical Definition:**
The three-part framework produced by ODMG for standardizing object databases.

**Core Principles:**

```
ODMG Standard
    │
    ├── ODM (Object Data Model)
    │       └── Conceptual model: objects, classes, interfaces
    │
    ├── ODL (Object Definition Language)
    │       └── Schema definition language (like DDL in SQL)
    │
    └── OQL (Object Query Language)
            └── Query language (like SELECT in SQL)
```

**Exam Context:**
The mapping is:
- ODL ≈ CREATE TABLE in SQL
- OQL ≈ SELECT...FROM...WHERE in SQL
- ODM ≈ Relational model (the underlying theory)

---

### **[161] ODMG Object Model Components**

**Technical Definition:**
The building blocks of the ODMG model: objects, literals, interfaces, and classes.

**Core Principles:**

**[165] Interface — Non-Instantiable [165]:**
- Specifies *only behavior* (operations/methods).
- Cannot create instances directly.
- Used for defining shared behavior contracts.
- Example: A `Printable` interface specifying a `print()` operation.

**[166] Class — Instantiable:**
- Specifies *both state (attributes)* and *behavior (operations)*.
- Can create instances.
- Example: `Person` class with `name`, `address` attributes and `getAge()` method.

**[164] Behavior vs. State:**
- **Behavior** = operations (what the object can *do*)
- **State** = properties (what the object *has* — attributes and relationships)

---

## 4. Object Identity (OID)

### **[70] [147] Object Identity (OID)**

**Technical Definition:**
A unique, system-generated identifier assigned to every object at creation time, independent of the object's attribute values. Two objects can have identical attribute values but remain distinct due to different OIDs.

**Core Principles:**

This is the most fundamental departure from the relational model:

```
Relational Model:         Object Model:
─────────────────         ─────────────
Identity = primary key    Identity = OID
(value-based)             (value-independent)

Two tuples with same      Two objects with same
primary key = SAME tuple  attributes = STILL DISTINCT
```

**Why OID Matters:**
1. Allows **sharing** — multiple objects can reference the same object via its OID.
2. Supports **mutable objects** — you can change all attributes; the object is still the same object.
3. Enables **object graphs** — networks of inter-referencing objects.

**[93] Four Degrees of OID Permanence:**

| Level | Scope | Description |
|-------|-------|-------------|
| **[94] Intraprocedure** | Single procedure | Like a local variable; identity gone when procedure returns |
| **[95] Intraprogram** | Single program/query | Like a pointer in memory; gone when program ends |
| **[96] Interprogram** | Across programs | Survives program restart, but not data reorganization |
| **[97] Persistent** | Permanent | Survives data reorganization; true OID |

**Exam Context:**
- A database requires **persistent** identity.
- In-memory OO languages typically provide only intraprocedure or intraprogram identity.
- The whole point of persistent programming languages is to elevate OID to the **persistent** level.

**Practical Example:**
```
Object: Employee #OID-4892
    name = "Alice"
    salary = 50000

After promotion:
Object: Employee #OID-4892   ← SAME object, SAME OID
    name = "Alice"
    salary = 75000
```
In a relational system, identity is `name = 'Alice'` — changing the primary key would "lose" the identity.

---

## 5. Complex Objects & Composition

### **[148] Complex Objects — Composition**

**Technical Definition:**
The ability for objects to contain other objects as attribute values, creating hierarchical or graph-structured data. This is known as **composition** or **aggregation** in OO terminology.

**Core Principles:**

Three levels of composition:

```
Level 1 — Simple Object:
    Person { name: String, age: Integer }

Level 2 — Composed Object:
    Person { name: Name, address: Address }
    where Name = { first: String, last: String }

Level 3 — Deeply Nested:
    Department { 
        name: String, 
        head: Person,        ← reference to another object
        members: Set<Person> ← collection of objects
    }
```

**Why Composition Is Powerful:**
- Models real-world structures directly without decomposition.
- No need for joins to reassemble related data.
- Objects naturally form a graph, not just a flat table.

**Exam Context:**
Composition (contains) vs. Reference (points to):
- A `Book` that *contains* an `Author` object = composition.
- A `Department` that *references* a `Person` (head) = reference/association.

---

## 6. Classes, Interfaces & Inheritance

### **[149] Class Hierarchies**

**Technical Definition:**
The organization of object types into a hierarchy where subclasses inherit attributes and operations from superclasses.

**Core Principles:**

**[167] Behavior Inheritance (IS-A) — Colon Notation:**
```
interface Printable {
    print(): void
};

interface Document : Printable {   // Inherits print()
    getTitle(): string
};
```
- Also called **interface inheritance**.
- A subinterface IS-A superinterface.
- Inherits only behavior (operations), not state.

**[168] EXTENDS Inheritance — State + Behavior:**
```
class Person {
    attribute string name;
    attribute string address;
};

class Student extends Person {   // Inherits name, address
    attribute string degree;
    attribute string department;
};
```
- Inherits **both state (attributes) and behavior (operations)**.
- Strictly between **classes** (not interfaces).

**[169] Single EXTENDS Constraint:**
- **Multiple inheritance via `extends` is NOT permitted** in ODMG.
- Multiple interface inheritance (via `:`) IS permitted.
- This avoids the "diamond problem" for state inheritance.

**Exam Trap:**
Students often confuse the two inheritance mechanisms:

| Mechanism | Keyword | Inherits | Multiple? |
|-----------|---------|----------|-----------|
| IS-A | `:` (colon) | Behavior only | ✅ Yes |
| EXTENDS | `extends` | State + Behavior | ❌ No |

---

## 7. Encapsulation & Methods

### **[152] Method Encapsulation**

**Technical Definition:**
The bundling of data (attributes) and the operations that act on that data (methods) together within an object. External code interacts with the object only through defined methods, not by directly accessing internal state.

**Core Principles:**

```
Without Encapsulation:         With Encapsulation:
──────────────────────         ────────────────────
External code directly         External code calls
reads/writes attributes        methods only

person.age = -5;  // BUG!     person.setAge(-5);
                               // Method validates before setting
```

**Benefits:**
1. **Data hiding** — internal representation can change without breaking external code.
2. **Validation** — methods enforce invariants.
3. **Reusability** — behavior travels with the object.
4. **Maintainability** — changes localized to one place.

**[177] Operation Signature:**
Every method has a signature: `operation_name(arg_type1, arg_type2, ...) : return_type`

Example:
```
class Person {
    attribute string name;
    attribute date dateOfBirth;
    
    // Signature: ageOnDate(date) : interval
    ageOnDate(onDate: date): interval year;
};
```

---

## 8. Object References & Path Expressions

### **[150] Direct Object References**

**Technical Definition:**
Objects can store references (OID-based pointers) to other objects. This is the primary mechanism for expressing relationships between objects in an OODBMS.

**Core Principles:**

In SQL object-relational systems, references use `ref(Type)`:

```sql
-- Define a type with a reference
create type Department (
    name    varchar(20),
    head    ref(Person) scope people   -- reference to a Person
);
```

**Reference vs. Foreign Key:**

| | Foreign Key (RDBMS) | Reference (OODB) |
|---|---|---|
| Based on | Attribute value | OID |
| Navigated via | JOIN | `->` operator |
| Type-safe? | No (just a value) | Yes (typed ref) |
| Can dangle? | Via constraints | Via scope clause |

---

### **[75] [76] Path Expressions and `->` Operator**

**Technical Definition:**
A path expression navigates through a chain of object references to reach a desired attribute or method. The `->` operator dereferences a reference to access the referenced object's properties.

**Core Principles:**

```sql
-- Without path expressions (requires JOIN):
select p.name, p.address
from departments d, people p
where d.name = 'CS' and d.head_id = p.person_id;

-- With path expressions (no JOIN needed):
select head->name, head->address
from departments;
```

**Multi-Hop Path Expressions:**
```
department->head->address->city
     │         │       │       │
  reference  Person  Address  String
```

**[77] Avoiding Explicit Joins:**
This is the key practical advantage. In deeply nested object graphs, a single path expression replaces multiple joins.

**[192] OQL Path Expression Orthogonality:**
In OQL, path expressions can traverse **attributes**, **relationships**, AND **method calls** interchangeably:
```
// All valid in OQL path expressions:
person.name           // attribute
person.worksFor       // relationship
person.getAge()       // method call
```

**Exam Context:**
"What eliminates the need for explicit joins in OODBMS?" → **Path expressions** using the `->` operator.

---

## 9. Object Persistence

### **[151] Object Persistence**

**Technical Definition:**
The ability of objects to survive beyond the execution of the program that created them, being stored in non-volatile storage (disk) and retrievable in future program executions.

**Core Principles:**
Persistence is what distinguishes a *database object* from an *in-memory object*.

**[89–92] Four Approaches to Persistence:**

| Approach | Mechanism | Example |
|----------|-----------|---------|
| **[89] Persistence by Class** | Declare a class as persistent; all its instances are persistent | `persistent class Person {...}` |
| **[90] Persistence by Creation** | Use special syntax when creating the object | `new(database) Person(...)` |
| **[91] Persistence by Marking** | Create normally, then explicitly mark as persistent | `object.makePersistent()` |
| **[92] Persistence by Reachability** | Object is persistent if reachable from a persistent root | Most common in Java/JDO |

**Persistence by Reachability — Detailed:**
```
Root (persistent) ──→ Object A (becomes persistent)
                           └──→ Object B (also becomes persistent)
                                     └──→ Object C (also persistent)
```
Any object reachable from the persistent root automatically becomes persistent. Objects not reachable are transient and will be garbage collected.

**Exam Context:**
- JDO (Java) uses **persistence by reachability**.
- ODMG C++ offers multiple mechanisms including **persistence by creation** (`new(db) T()`).

---

## 10. Persistent Programming Languages (C++ & Java)

### **[87] Persistent Programming Languages**

**Technical Definition:**
Programming languages extended with constructs to handle persistent data — allowing programmers to work with database-resident objects using the same syntax as in-memory objects, without explicit fetch/store operations.

**Core Principles:**

**[88] Advantage — Direct Persistent Data Manipulation:**
```
Traditional (Embedded SQL):           Persistent Language:
──────────────────────────            ────────────────────
EXEC SQL SELECT name                  string name = person->name;
    INTO :name FROM people            // Just use it — transparent!
    WHERE id = :id;
name = process(name);
EXEC SQL UPDATE people
    SET name = :name
    WHERE id = :id;
```

**[98] ODMG C++ — Key Features:**

| Feature | Description | Syntax |
|---------|-------------|--------|
| **[101] Persistent Pointers** | Type-safe persistent references | `d_Ref<Person> head;` |
| **[102] Object Creation** | Create object in persistent store | `new(db) Person(...)` |
| **[103] Class Extents** | Access all persistent objects of a type | Via extent iterator |
| **[104] Relationships** | Pointer-based with consistency | Bidirectional refs |
| **[105] Back-references** | Auto-maintained reverse pointers | Prevents dangling refs |
| **[107] mark_modified()** | Signal object was updated | `obj->mark_modified()` |
| **Transactions** | ACID transactions | `db->begin_transaction()` |

**Why `mark_modified()` Is Needed:**
The system cannot know if a fetched object was modified (C++ allows direct attribute access). The programmer must explicitly tell the system: "I changed this object; write it back."

---

### **[100] Java Database Objects (JDO)**

**Technical Definition:**
The standard approach for adding persistence to Java objects, allowing POJOs (Plain Old Java Objects) to be stored in and retrieved from a database.

**Core Principles:**

**[108] Byte Code Enhancement:**
```
Original .java file → Compile → .class file
                                    ↓
                              Byte code enhancer
                                    ↓
                           Enhanced .class file
                           (with persistence hooks)
```
The bytecode modifier adds:
- Code to **fetch from DB** when object is first accessed.
- Code to **mark object dirty** when attributes are written.

**[109] Fetch on Demand (Lazy Loading):**
Objects are not loaded from the DB until actually accessed. This avoids loading entire object graphs when only a small part is needed.

**[110] Hollow Objects (Pointer Swizzling):**
When a persistent reference is followed:
1. Object exists in memory as a "hollow" placeholder (just the OID).
2. On first attribute access, the hollow object is "filled" from DB.
3. In-memory pointer is updated to point to the real object.

**[106] Single Reference Type:**
No distinction between in-memory pointer and persistent reference — the programmer uses the same `obj.attribute` syntax for both. The system handles the difference transparently.

**Exam Context:**
- JDO uses **persistence by reachability** — mark a root object persistent; all connected objects persist.
- Unlike ODMG C++, JDO has a **single reference type** (no `d_Ref<T>`).

---

## 11. Object Query Language (OQL)

### **[7] OQL Overview**

**Technical Definition:**
OQL is the ODMG standard query language for object databases, providing SQL-like SELECT-FROM-WHERE syntax while operating on objects, collections, and OIDs rather than tables and rows.

**Core Principles:**

**OQL is:**
- **Declarative** — specifies *what*, not *how*.
- **Not computationally complete** — cannot express all algorithms (intentionally, like SQL).
- **Syntactically based on SQL** — familiar SELECT-FROM-WHERE.
- **Extended** for OO features — method calls, path expressions, object collections.

---

### **[189] OQL Entry Points**

**Technical Definition:**
Named persistent objects that serve as the starting point for OQL queries. Since there are no "tables" in a pure OODBMS, queries must begin from a named extent or a named object.

**Core Principles:**
```
// In SQL:
SELECT * FROM products WHERE color = 'black';
//             ^^^^^^^^ table name

// In OQL:
SELECT DISTINCT p.name
FROM products p            ← 'products' is a named extent/collection
WHERE p.color = "black";
```

**[190] OQL Iterator Variables:**
An iterator variable is defined for every collection referenced in a FROM clause:
```
FROM products p    ← 'p' is the iterator variable
                     ranging over all objects in 'products'
```

---

### **[191] OQL Query Result Types**

**Technical Definition:**
OQL queries can return any type expressible in the ODMG object model — collections of objects, single objects, scalar values, or structured types.

**Core Principles:**
Unlike SQL which always returns a table:
```
SELECT p.name              → returns a Bag<string>
FROM products p
WHERE p.color = "black"

SELECT p                   → returns a Bag<Product>
FROM products p
WHERE p.color = "black"

element(                   → returns a single Product
  SELECT p
  FROM products p
  WHERE p.productNo = "P1"
)
```

**[194] `element` Operator:**
Used when a query is expected to return exactly one object. Raises an error if zero or more than one object is returned — enforces single-result queries.

---

### **[195] OQL Aggregate Operators**

**Technical Definition:**
OQL supports SQL-style aggregate operations plus quantifiers for reasoning over collections.

**Core Principles:**

**Standard Aggregates:** `count()`, `sum()`, `avg()`, `min()`, `max()`

**Quantification Operators (not in SQL):**
```
// Existential: does any element satisfy the condition?
exists p in products: p.color = "black"

// Universal: do all elements satisfy the condition?
for all p in products: p.price > 0

// Membership:
"black" in (select p.color from products p)
```

---

### **[196] Special Operations for Ordered Collections**

**Technical Definition:**
OQL provides additional operations for **list** and **array** types where element order is meaningful.

**Core Principles:**
- `first(collection)` — first element
- `last(collection)` — last element
- `collection[i]` — element at position i
- `flatten(collection)` — flatten nested collections

---

### **[197] OQL GROUP BY**

**Technical Definition:**
Similar to SQL GROUP BY, but OQL provides **explicit access to the partition** (the collection of objects within each group).

**Core Principles:**
```sql
-- SQL GROUP BY (you only get aggregates):
SELECT dept, COUNT(*) FROM employees GROUP BY dept;

-- OQL GROUP BY (you get the partition object):
SELECT dept, count(partition)
FROM employees
GROUP BY dept: e.department
```
The keyword `partition` refers to the entire collection of objects within the current group — allowing more complex group-level operations.

**[198] HAVING Clause in OQL:**
Filters groups after grouping, identical in concept to SQL:
```
SELECT dept, count(partition)
FROM employees
GROUP BY dept: e.department
HAVING count(partition) > 5
```

---

### **[193] Named Queries**

**Technical Definition:**
OQL allows queries to be given names and stored as database objects, making them reusable entry points.

```
define BlackProducts as
    select p from products p where p.color = "black"

// Later, query the named result:
select bp.name from BlackProducts bp
```

---

## 12. OQL vs SQL — Key Differences

### **[154] Relational vs. OO Result Differences**

**Technical Definition:**
Despite nearly identical syntax, SQL and OQL queries against equivalent datasets return fundamentally different result structures.

---

### **[155] SQL Returns Tables**

A SQL query result is always a **table** (a set of rows with named columns):

```
Query: SELECT DISTINCT p.name FROM products p WHERE p.color = 'black'

SQL Result:
┌──────────────────┐
│ Name             │
├──────────────────┤
│ Ford Mustang     │
│ Mercedes SLK     │
└──────────────────┘
(a single table with 2 rows)
```

---

### **[156] OQL Returns Object Collections**

An OQL query result is a **collection of objects** (each a first-class object with its own identity):

```
OQL Result:
┌────────────────┐    ┌────────────────┐
│ String         │    │ String         │
│ "Ford Mustang" │    │ "Mercedes SLK" │
└────────────────┘    └────────────────┘
(two separate String objects in a Bag)
```

**[157] Table Name vs. Collection Name:**
- SQL: FROM clause refers to a **table** (flat set of tuples)
- OQL: FROM clause refers to a **collection object** (containing typed objects)

**[158] Column Name vs. Characteristic Name:**
- SQL: column names refer to **attribute values** only
- OQL: names can refer to **attributes, relationships, or method results** interchangeably

---

### **[22–23] Comparison Summary Table**

| Feature | SQL / RDBMS | OQL / OODBMS |
|---------|------------|--------------|
| Result type | Table (rows) | Collection of objects |
| Identity | Primary key (value) | OID (system-generated) |
| Navigation | JOINs | Path expressions (`->`) |
| Nested data | Multiple tables + joins | Composition |
| Behavior | Stored procedures (separate) | Methods (encapsulated) |
| Inheritance | Not native | IS-A and EXTENDS |
| Collections | Limited (arrays in SQL:1999) | First-class (set, list, bag, array, dict) |

---

## 13. Extents, Keys & Factory Objects

### **[178] Extent**

**Technical Definition:**
The **extent** of a class is the set of all currently existing persistent instances of that class. It is the OO equivalent of a table in the relational model.

**Core Principles:**
```
Class Person       ←  Type definition (like a schema)
    │
    ▼
Extent of Person   ←  All persistent Person objects (like table contents)
    { OID-001: {name:"Alice", age:30},
      OID-002: {name:"Bob",   age:25},
      OID-003: {name:"Carol", age:22} }
```

- Not all classes need extents (transient objects have no extent).
- Extent is explicitly declared in ODL.
- Enables class-wide queries: "find all Persons where age > 25".

**Exam Context:**
A class without an extent cannot be queried globally — you can only navigate to its instances via references from other objects.

---

### **[179] Key in Extents**

**Technical Definition:**
A **key** in the context of extents is one or more attributes whose combined values are unique across all objects in the extent. It is similar to a primary key in a relational table, but applies to an object collection.

**Core Principles:**
```
class Person
    (extent people
     key name)           // 'name' is unique across all Person objects
{
    attribute string name;
    attribute string address;
};
```

- A key **does not replace** the OID — the OID is always the true identity.
- The key provides a **human-accessible unique lookup** path.
- Multiple keys can be defined: `key (firstName, lastName)`.

**Exam Context:**
Key vs. OID:
- **OID**: system-generated, immutable, user never sees it directly.
- **Key**: user-visible attribute(s), can theoretically change (with constraints).

---

### **[180] Factory Object**

**Technical Definition:**
A factory object is an object whose purpose is to **create** (instantiate) other objects of a specific class. It provides a controlled interface for object creation.

**Core Principles:**
```
// Factory pattern in ODMG:
factory PersonFactory {
    Person createPerson(name: string, address: string);
};

// Usage:
Person p = factory.createPerson("Alice", "123 Main St");
```

**Why Use Factories?**
1. Centralize object creation logic.
2. Validate inputs before creating objects.
3. Manage OID assignment.
4. Support object pooling or caching.

---

## 14. ODL — Object Definition Language

### **[35] ODL Overview**

**Technical Definition:**
ODL is the schema definition language of the ODMG standard. It defines classes, attributes, relationships, and operations independently of any programming language, serving as the DDL (Data Definition Language) of the ODMG framework.

**Core Principles:**

**[181] Language Independence:**
ODL is designed to map to multiple programming languages:
- ODL → C++ binding
- ODL → Java binding (JDO)
- ODL → Smalltalk binding

**Basic ODL Structure:**
```
class ClassName
    (extent extentName       // optional: names the collection of all instances
     key keyAttribute)       // optional: declares unique key
{
    // Attributes (state):
    attribute type attributeName;

    // Relationships (connections to other objects):
    relationship Type relationshipName
        inverse OtherClass::inverseRelationship;

    // Operations (behavior):
    returnType operationName(argType argName, ...);
};
```

---

### **[174] Attribute Property**

**Technical Definition:**
An attribute in ODL describes some aspect of an object's state. It maps to a value, not to another object (for object-to-object connections, use relationships).

```
attribute string    name;
attribute date      dateOfBirth;
attribute float     salary;
```

---

### **[175] [176] Relationship Property & `inverse` Keyword**

**Technical Definition:**
A relationship in ODL connects two objects. Every relationship must have an **inverse** — the other direction of the same conceptual connection, declared using the `inverse` keyword.

**Core Principles:**
```
class Department {
    relationship Set<Person> members
        inverse Person::worksIn;      // inverse direction
};

class Person {
    relationship Department worksIn
        inverse Department::members;  // inverse direction
};
```

**Why Inverse?**
- Ensures **referential integrity** automatically.
- When you add a Person to `Department.members`, the system automatically sets that Person's `worksIn` to this Department.
- No need for manual bidirectional updates.

**Exam Context:**
A relationship without `inverse` is just an attribute that happens to hold a reference — it doesn't get automatic consistency maintenance.

---

## 15. Conceptual Design — EER to ODB Mapping

### **[182] ODB vs. RDB Design Differences**

**Technical Definition:**
Designing schemas for object databases differs from relational design in three key ways: handling relationships, inheritance, and behavioral specification.

**Core Principles:**

| Design Aspect | Relational (EER → ER → Relations) | Object (EER → ODL Classes) |
|--------------|----------------------------------|---------------------------|
| Relationships | Foreign keys + JOIN tables | Relationship properties + `inverse` |
| Inheritance | Separate tables per subclass | `extends` or `:` in ODL |
| Behavior | Stored procedures (separate) | Methods inside class definition |

---

### **[183] EER to ODB Mapping — Step by Step**

**[183] Step 1:** Create an ODL class for each EER entity type.
```
EER Entity: EMPLOYEE → ODL class Employee { ... }
```

**[184] Step 2:** Add relationship properties for each binary relationship.
```
EER Relationship: EMPLOYEE works-in DEPARTMENT
→ 
class Employee { relationship Department worksIn inverse Department::employees; }
class Department { relationship Set<Employee> employees inverse Employee::worksIn; }
```

**Step 3:** Include appropriate operations for each class.
```
class Employee { float computeTax(); }
```

**[185] Step 4:** Map subclasses using ODL inheritance.
```
EER: MANAGER is a subclass of EMPLOYEE
→
class Manager extends Employee { attribute float budget; }
```

---

### **[186] Weak Entity Type Mapping**

**Technical Definition:**
Weak entity types (those depending on a strong entity for identification) are mapped to ODL classes just like regular entity types.

**Core Principles:**
- The identifying relationship becomes an ODL relationship.
- The partial key becomes an attribute.
- Full identity provided by OID (not by the key alone).

```
EER: DEPENDENT (weak, identified through EMPLOYEE)
→
class Dependent {
    attribute string name;     // partial key
    relationship Employee dependentOf
        inverse Employee::dependents;
};
```

---

### **[187] Category (Union Type) Mapping**

**Technical Definition:**
A category (or union type) in EER represents an entity that can be of one of several types (e.g., an OWNER can be a PERSON or a COMPANY). These are **difficult to map directly to ODL**.

**Core Principles:**
Challenges:
- ODL inheritance is single-root (one superclass per extends chain).
- A union type requires an entity to have different superclasses in different cases.

Workaround approaches:
1. Use a common interface that both PERSON and COMPANY implement.
2. Use an OID reference in the category class pointing to either a PERSON or COMPANY object.

**Exam Context:**
"Why are categories (union types) difficult to map to ODL?" → ODL's inheritance model doesn't natively support the union/OR-specialization semantics of EER categories.

---

### **[188] N-ary Relationship Mapping**

**Technical Definition:**
A relationship involving more than two entity types (degree n > 2) is mapped to a **separate class** in ODL, with reference attributes pointing to each participating class.

**Core Principles:**
```
EER: EMPLOYEE works-on PROJECT at LOCATION (ternary)
→
class WorksOn {
    relationship Employee  employee  inverse Employee::assignments;
    relationship Project   project   inverse Project::assignments;
    relationship Location  location  inverse Location::assignments;
    attribute integer hoursPerWeek;  // relationship attribute
};
```

This mirrors the relational approach of creating a junction/association table for n-ary relationships.

---

## 16. Object-Relational Mapping (ORM)

### **[111] Object-Relational Mapping (ORM)**

**Technical Definition:**
ORM systems provide a layer that maps between an object model (used by application programmers) and a relational model (used for persistent storage), allowing developers to work with objects while data is stored in a relational database.

**Core Principles:**

**[142] Transient Object Model:**
- Objects in ORM are **purely transient** — they exist only in application memory.
- There is **no permanent object identity** — OID is simulated via primary key mapping.
- Objects are created from relational data and converted back for storage.

```
Application Layer        ORM Layer              Database Layer
─────────────────        ─────────────          ──────────────
Person object     ←──→  Mapping rules   ←──→   PERSON table
  name                   (XML/annotation)        name column
  address                                        address column
  getAge()                                       (no method)
```

**[143] Mapping Objects to Relations:**
The developer provides a mapping (in XML or annotations):
```java
@Entity
@Table(name = "PERSON")
public class Person {
    @Id
    @Column(name = "PERSON_ID")
    private int id;
    
    @Column(name = "NAME")
    private String name;
}
```

**[144] Object Retrieval from Relations:**
When you load an object, ORM:
1. Issues a SQL SELECT.
2. Maps result columns to object attributes.
3. Creates and returns the object.

**[145] Update Generation:**
When you save a modified object, ORM:
1. Detects changed attributes (via dirty tracking).
2. Generates appropriate INSERT/UPDATE/DELETE SQL.
3. Executes against the database.

**[113] Hibernate ORM:**
The most widely used Java ORM framework:
- Provides API for `beginTransaction()`, `commit()`, `get()`, `save()`.
- HQL (Hibernate Query Language) — object-oriented query language.
- HQL queries are translated to SQL automatically.

**[115] ORM Limitations:**
- **Performance overhead** — especially for bulk updates (generates one UPDATE per object).
- **Object-relational impedance mismatch** — not all OO structures map cleanly to tables.
- **No true identity** — simulated via primary key, not a real OID.

---

## 17. System Comparisons

### **[38 / 44] Comprehensive Comparison of Database Paradigms**

| Feature | Relational [116/199-201] | OODBMS [117/202-204] | Object-Relational [118/205-207] | ORM [119/208] |
|---------|---------|-------|----------------|-----|
| **Data Types** | Simple atomic | Complex, nested | Complex + relational | Complex (in OO layer) |
| **Query Language** | SQL (powerful) | OQL (powerful) | Extended SQL | OQL over SQL |
| **Performance** | High for set ops | High for navigation | Medium | Lower (overhead) |
| **Integrity/Protection** | High | Medium | High | Depends on RDBMS |
| **OO Integration** | None | Full | Partial | Full (in app layer) |
| **Standards** | SQL (mature) | ODMG (limited adoption) | SQL:1999 | JPA/Hibernate |
| **Industry Adoption** | Very High | Low | High | High (app layer) |

---

### **[209] System Boundary Blurring**

**Technical Definition:**
In practice, the four paradigm categories (Relational, OODB, OR, ORM) are not cleanly separated. Many real systems combine features from multiple paradigms.

**Core Principles:**
Examples of boundary blurring:
- A **persistent programming language** (OODBMS) built as a **wrapper over a relational database** gains both OO integration AND SQL storage — but may sacrifice performance.
- **PostgreSQL** supports user-defined types, arrays, JSON, table inheritance — making it closer to ORDBMS than pure RDBMS.
- **MongoDB** (document store) has some OO characteristics without being a true OODBMS.

**Exam Context:**
If asked to classify a real system, always discuss what features it shares from each paradigm rather than trying to force it into one category.

---

## Quick Concept Summary by Number

| Concept # | Concept Name | Section |
|-----------|-------------|---------|
| 1 | RDBMS Limitations | §1 |
| 2 | Complex Data Applications | §1 |
| 3 | Object-Database Systems | §1 |
| 4 | OODBMS | §2 |
| 5 | ODMG | §3 |
| 6 | Object Data Model | §3 |
| 7 | OQL | §11 |
| 10 | OODBMS Historical Failure | §1 |
| 11 | Object-Relational Success | §1 |
| 70 | Object Identity | §4 |
| 75 | Path Expressions | §8 |
| 76 | `->` Operator | §8 |
| 87–92 | Persistence Approaches | §9 |
| 93–97 | OID Permanence Levels | §4 |
| 98–110 | Persistent C++ / JDO | §10 |
| 111–115 | ORM Systems | §16 |
| 146–152 | OODBMS Core Features | §2, §4–§8 |
| 153 | ODMG Components | §3 |
| 154–158 | OQL vs SQL | §12 |
| 161–169 | ODMG Object Model | §3, §6 |
| 170–172 | Collections | §13 |
| 173–177 | ODL Constructs | §14 |
| 178–180 | Extents, Keys, Factories | §13 |
| 181–188 | ODL Design & Mapping | §14, §15 |
| 189–198 | OQL Features | §11 |
| 199–209 | System Comparisons | §17 |

---

## Key Exam Traps — OO Database Edition

1. **OID ≠ Primary Key**: OID is system-generated and value-independent. Primary key is a user-chosen value.

2. **`extends` ≠ `:` (colon)**: `extends` inherits state+behavior (single only); `:` inherits behavior only (multiple allowed).

3. **OQL result ≠ SQL result**: Same query syntax, but OQL returns objects, SQL returns rows.

4. **Persistence by reachability**: If object A (persistent) → B → C, then B and C are also persistent even without explicit declaration.

5. **Interface = non-instantiable**: You cannot `new Interface()`. You can only instantiate classes.

6. **Extent is optional**: A class can exist without an extent — you just won't be able to query all its instances.

7. **`inverse` is bidirectional**: Declaring `inverse` creates a single conceptual relationship in both directions — not two separate relationships.

8. **ORM has no true OID**: Hibernate uses primary key as surrogate — it's value-based identity, not true object identity.

9. **OODBMS failed for efficiency reasons**: Not because the data model was bad — the model was excellent. The query optimizer and declarative query language were inadequate.

10. **`element` operator enforces singleton**: Using `element()` on a result with 0 or 2+ objects raises an error.

---

*End of OO Database Concepts — CS G516, BITS-Pilani*
