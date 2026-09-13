# Database & Node.js ⭐⭐⭐

## Topics Covered

- [1. Database Connection](#1-database-connection)
- [2. Connection Pooling](#2-connection-pooling)
- [3. CRUD](#3-crud)
- [4. Transactions](#4-transactions)
- [5. Indexes](#5-indexes)
- [6. SQL vs NoSQL Basics](#6-sql-vs-nosql-basics)
- [7. ORM/ODM Basics](#7-ormodm-basics)
- [8. Query Optimization](#8-query-optimization)

---

## 1. Database Connection

You should understand:

- How Node.js connects to a database
- Connection lifecycle
- Connection strings
- Environment variables for DB credentials
- Handling connection errors
- Closing database connections
- Difference between opening a new connection for every request vs reusing
  connections

### Typical flow

```text
Node.js Application
       ↓
Database Driver / ORM
       ↓
Connection
       ↓
Database
```

### Interview questions

1. How does Node.js connect to a database?
2. Where should database credentials be stored?
3. What happens if the database connection fails?
4. Should you create a new DB connection for every API request?
5. How do you handle database connection errors?

---

## 2. Connection Pooling ⭐⭐⭐

This is very important for interviews.

Instead of creating a new database connection for every request, Node.js
maintains a pool of reusable connections.

```text
             ┌── Connection 1
             ├── Connection 2
Node.js ─────┼── Connection 3
             ├── Connection 4
             └── Connection 5
                 ↓
              Database
```

When a request needs the database:

```text
Request
  ↓
Get available connection
  ↓
Execute query
  ↓
Release connection
  ↓
Connection returns to pool
```

### Why pooling?

- Better performance
- Avoids connection creation overhead
- Handles concurrent requests
- Prevents too many database connections
- Reuses existing connections

### Interview questions

1. What is connection pooling?
2. Why do we need connection pooling?
3. What happens when all pool connections are busy?
4. How do you configure pool size?
5. What happens if you don't release a connection?

Very common interview question:

> **Why shouldn't we create a new database connection for every request?**

Because establishing connections is expensive and can overwhelm the
database under high traffic. A connection pool allows connections to be
reused.

---

## 3. CRUD ⭐⭐⭐

CRUD means:

| Operation | SQL | HTTP |
| --- | --- | --- |
| Create | `INSERT` | `POST` |
| Read | `SELECT` | `GET` |
| Update | `UPDATE` | `PUT`/`PATCH` |
| Delete | `DELETE` | `DELETE` |

Example:

```sql
INSERT INTO users (name, email)
VALUES ('John', 'john@example.com');

SELECT * FROM users;

UPDATE users
SET name = 'Mike'
WHERE id = 1;

DELETE FROM users
WHERE id = 1;
```

With Node.js, you'll typically use:

```text
Node.js
   ↓
Driver / ORM
   ↓
SQL query
   ↓
Database
```

### Important interview concepts

- Parameterized queries
- SQL injection
- `WHERE` conditions
- `JOIN`
- Pagination
- Sorting
- Filtering
- Selecting only required columns

### Important security point

Avoid:

```js
const query = `SELECT * FROM users WHERE id = ${id}`;
```

Prefer parameterized queries:

```js
const query = "SELECT * FROM users WHERE id = ?";
```

This helps prevent SQL injection.

**Note:** placeholder syntax differs by driver — it isn't universal.

- **MySQL** (`mysql`/`mysql2`) uses positional `?` placeholders:

  ```js
  connection.query("SELECT * FROM users WHERE id = ?", [id]);
  ```

- **PostgreSQL** (the `pg` driver) uses numbered `$1`, `$2`, ... placeholders
  instead:

  ```js
  client.query("SELECT * FROM users WHERE id = $1", [id]);
  ```

Whichever driver/database you're using, the key point for interviews is
the same: never concatenate user input into a query string — always let
the driver bind parameters.

---

## 4. Transactions ⭐⭐⭐

A transaction groups multiple database operations into a single logical
operation.

Example: transferring money.

```text
Account A
   ↓
Deduct ₹1000
   ↓
Account B
   ↓
Add ₹1000
```

Both operations should succeed, or neither should happen.

```sql
BEGIN TRANSACTION

Deduct money
Add money

COMMIT
```

If something fails:

```sql
BEGIN TRANSACTION

Deduct money
Add money ❌

ROLLBACK
```

### ACID

You should definitely know this for interviews.

**A — Atomicity**

All operations succeed or all fail.

**C — Consistency**

Database remains in a valid state.

**I — Isolation**

Concurrent transactions don't improperly interfere with each other.

Isolation is enforced through **isolation levels**, which trade off
consistency against concurrency/performance:

| Level | Prevents |
| --- | --- |
| Read Uncommitted | Nothing — allows dirty reads |
| Read Committed | Dirty reads |
| Repeatable Read | Dirty reads, non-repeatable reads |
| Serializable | Dirty reads, non-repeatable reads, phantom reads |

What each problem means:

- **Dirty read** — reading data that another transaction has written but
  not yet committed (and might roll back).
- **Non-repeatable read** — reading the same row twice in one transaction
  and getting different values because another transaction updated and
  committed it in between.
- **Phantom read** — re-running the same query twice in one transaction
  and getting a different **set of rows** because another transaction
  inserted/deleted matching rows in between.

Higher isolation levels prevent more of these anomalies but generally
reduce concurrency (more locking, more blocked/retried transactions).

**D — Durability**

Once committed, data survives failures.

### Deadlocks

A **deadlock** happens when two (or more) transactions each hold a lock the
other needs, and each is waiting for the other to release it — neither can
proceed.

```text
Transaction A: locks Row 1, waits for Row 2
Transaction B: locks Row 2, waits for Row 1
```

Databases detect this (often via a wait-for graph or a lock-wait timeout)
and resolve it by picking a **"victim"** transaction to abort and roll
back, letting the other proceed. Applications should be prepared to catch
a deadlock error and retry the aborted transaction. Consistently locking
resources in the same order across your codebase reduces how often
deadlocks occur.

### Optimistic vs pessimistic locking

Two common strategies for handling concurrent updates to the same data:

- **Pessimistic locking** — lock the row upfront (e.g., `SELECT ... FOR
  UPDATE`) before reading/modifying it, so no other transaction can touch
  it until you're done. Safer under heavy contention, but reduces
  concurrency since other transactions must wait.
- **Optimistic locking** — don't lock anything upfront. Instead, read the
  data along with a version number (or timestamp), and when writing back,
  check that the version hasn't changed (`WHERE id = ? AND version = ?`).
  If it has changed, someone else updated it first — reject/retry. Better
  performance when conflicts are rare, since nothing blocks readers.

Rule of thumb: use optimistic locking when conflicts are uncommon, and
pessimistic locking when contention on the same rows is frequent.

### Interview questions

1. What is a transaction?
2. Why do we need transactions?
3. What is `COMMIT`?
4. What is `ROLLBACK`?
5. Explain ACID.
6. Give a real-world example of a transaction.

---

## 5. Indexes ⭐⭐⭐

Indexes improve database query performance.

Without an index:

```text
Query
 ↓
Scan entire table
 ↓
Find record
```

With an index:

```text
Query
 ↓
Index
 ↓
Find record faster
```

For example:

```sql
SELECT * FROM users
WHERE email = 'john@example.com';
```

If `email` is frequently searched, an index can help:

```sql
CREATE INDEX idx_users_email
ON users(email);
```

### But indexes aren't free

Indexes:

- Improve reads
- Consume storage
- Make inserts/updates/deletes somewhat more expensive
- Need to be chosen carefully

### Interview questions

1. What is a database index?
2. Why does an index improve performance?
3. What are the disadvantages of indexes?
4. When should you create an index?
5. What happens if you index every column?
6. What is a composite index?

### Important

You should understand:

- Index
- Composite Index
- Unique Index
- Primary Key

---

## 6. SQL vs NoSQL Basics ⭐⭐⭐

You don't need deep database administration knowledge, but you should
understand the fundamental difference.

### SQL

Examples:

- PostgreSQL
- MySQL
- SQL Server
- Oracle

Data is generally organized into tables and rows.

```text
Users
-------------------
id | name | email
1  | John | ...
2  | Mike | ...
```

Good when you need:

- Relationships
- Strong consistency
- Complex queries
- Transactions
- Structured data

### NoSQL

Examples:

- MongoDB
- Redis
- DynamoDB

Data models vary, commonly documents/key-value/etc.

MongoDB example:

```json
{
  "_id": 1,
  "name": "John",
  "email": "john@example.com"
}
```

Useful when:

- Data structure is flexible
- Horizontal scaling is important
- Document-oriented data fits the application
- You don't need relational joins for every operation

### Eventual consistency & CAP theorem

Many NoSQL databases favor horizontal scaling by distributing/replicating
data across multiple nodes. The **CAP theorem** says a distributed system
can only fully guarantee two of these three at once:

- **C — Consistency**: every read gets the most recent write.
- **A — Availability**: every request gets a (non-error) response.
- **P — Partition tolerance**: the system keeps working despite network
  partitions between nodes.

Since network partitions can always happen in a distributed system, the
real-world tradeoff is usually **C vs A**. Many NoSQL databases choose
**availability** over strict consistency, meaning a read right after a
write on a different node might return slightly stale data until
replication catches up — this is called **eventual consistency**: the
system guarantees that, given no new writes, all replicas will
*eventually* converge to the same value, just not instantly. Traditional
SQL databases more commonly prioritize strong consistency, sometimes at
the cost of availability during a partition.

### Interview question

**When would you choose SQL over MongoDB?**

Don't answer:

> SQL is better.

Instead:

> It depends on the application's data and access patterns. SQL is often
> preferable when relationships, complex queries, and transactional
> consistency are important. A document database can be a good fit when
> data is naturally document-oriented and the schema needs more
> flexibility.

---

## 7. ORM/ODM Basics ⭐⭐⭐

### ORM

**Object Relational Mapping.**

Used with relational databases.

Examples:

- Prisma
- Sequelize
- TypeORM

Instead of writing:

```sql
SELECT * FROM users WHERE id = 1;
```

you may write something like:

```js
const user = await prisma.user.findUnique({
  where: { id: 1 },
});
```

The ORM translates application operations into database queries.

### ODM

**Object Document Mapping.**

Usually used with document databases.

For example:

```text
MongoDB
   ↓
Mongoose
   ↓
Node.js
```

### Benefits

- Less boilerplate
- Models/schemas
- Type safety in some tools
- Query abstraction
- Relationships/associations
- Migrations in many ORMs

### Downsides

- Abstraction can hide inefficient queries
- Complex queries can be harder
- ORM overhead
- Developers still need database knowledge

### Interview questions

1. What is an ORM?
2. ORM vs ODM?
3. Why use Prisma/Sequelize/Mongoose?
4. Can an ORM replace SQL knowledge?
5. What are the disadvantages of ORMs?

Important interview answer:

> ORM knowledge doesn't replace database knowledge. You still need to
> understand queries, indexes, transactions, joins, and query performance
> because an ORM ultimately generates database operations.

---

## 8. Query Optimization ⭐⭐⭐

This is another high-value interview topic.

Suppose you have:

```sql
SELECT *
FROM users
WHERE email = 'john@example.com';
```

You should ask:

1. Is there an index on `email`?
2. Are we retrieving unnecessary columns?
3. Is the query scanning the whole table?
4. Is the query using a proper `WHERE` condition?
5. Are joins optimized?
6. How many rows are being returned?

Instead of:

```sql
SELECT *
FROM users;
```

prefer:

```sql
SELECT id, name, email
FROM users
LIMIT 20;
```

### Important optimization techniques

#### 1. Use indexes appropriately

```sql
CREATE INDEX idx_email ON users(email);
```

#### 2. Avoid unnecessary columns

Don't always use:

```sql
SELECT *
```

Select what you actually need.

#### 3. Pagination

Instead of returning 1 million records:

```http
GET /users
```

use:

```http
GET /users?page=1&limit=20
```

#### 4. Analyze queries

For SQL databases, understand the basic purpose of:

```sql
EXPLAIN
```

It helps inspect how the database plans to execute a query.

#### 5. Avoid N+1 queries

This is very important in Node.js interviews.

Bad pattern:

```text
Get 100 users
     ↓
Query database for each user's orders
     ↓
100 additional queries
```

Instead, use an appropriate:

```text
JOIN
```

or batch query/data-loading strategy.

---

## 🎯 What You Must Be Able to Explain

For this chapter, make sure you can answer these without notes:

### Database basics

1. How does Node.js connect to a database?
2. What is connection pooling?
3. Why is pooling important?
4. What is CRUD?
5. What is a transaction?
6. Explain ACID.
7. What are indexes?
8. Advantages/disadvantages of indexes.
9. SQL vs NoSQL.
10. When would you choose SQL vs NoSQL?
11. What is ORM?
12. What is ODM?
13. ORM vs ODM.
14. What is query optimization?
15. What is the N+1 query problem?
16. What is `EXPLAIN`?
17. How do indexes affect query performance?
18. How do you prevent SQL injection?
19. Why shouldn't an API return huge datasets?
20. How does pagination improve API/database performance?

## ⭐ Priority for interviews

If you're short on time, prioritize:

```text
1. Connection Pooling ⭐⭐⭐
2. Transactions + ACID ⭐⭐⭐
3. Indexes ⭐⭐⭐
4. SQL vs NoSQL ⭐⭐⭐
5. ORM/ODM ⭐⭐⭐
6. Query Optimization + N+1 ⭐⭐⭐
7. CRUD ⭐⭐
8. Database Connection ⭐⭐
```

This chapter is especially useful because interviewers often connect
Node.js + API + database + performance into one scenario rather than asking
each topic independently.
