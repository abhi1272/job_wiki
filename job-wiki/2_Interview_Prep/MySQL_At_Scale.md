# MySQL Database Scaling & Optimisation

This document covers techniques for scaling relational databases (MySQL/PostgreSQL) and resolving performance issues. It is grounded in database issues you resolved at ConnectWise (deadlock avoidance, query optimizations) and Concentrix.

[← Back to Index](../README.md) | [← Previous: Microservices Patterns](./Microservices_Patterns.md) | [Next: CI/CD Pipeline Flow](./CI_CD_Pipeline.md)

---

## 📈 Database Replication & Load Balancing

When database read traffic scales, a single database node becomes a CPU/Memory bottleneck. We scale out using replication:

```text
               [Write Traffic] ──> [MySQL Primary]
                                         │
                                   (Replication)
                                         ▼
[Read Traffic] ──> [ProxySQL] ──> [MySQL Replica Group]
```

### 1. Primary-Replica Architecture
* **Primary (Master):** Handles all write queries (`INSERT`, `UPDATE`, `DELETE`) and transaction blocks.
* **Replica (Slave):** Synchronizes database transactions from the primary asynchronously and serves read queries (`SELECT`).
* **Challenge:** Replication lag. Replicas might be slightly behind the primary. If a user writes data and immediately reads it (e.g., updating their profile name), they might read stale data.
* **Solution:** Configure your database driver to read sensitive data directly from the Primary.

### 2. ProxySQL Load Balancing
* **What:** A high-performance, protocol-aware proxy positioned between Node.js NestJS instances and MySQL databases.
* **How:** It parses incoming SQL queries in real-time. If the query starts with `SELECT`, ProxySQL routes it to a pool of Read Replicas. If it contains `INSERT` or is wrapped in a transaction block, it directs it to the Primary. It also handles connection pooling, SQL caching, and transparent failover.

---

## 📐 Indexing Strategies & Query Optimization

Indexes make query searches fast, but slow down writes because the index tree must be updated.

### 1. B-Tree Indexes
* **How it works:** MySQL's default InnoDB engine stores indexes as B+ Trees. Search complexity is `O(log N)` instead of `O(N)` full-table scans.
* **Composite Indexes:** When indexing multiple columns, order matters (Leftmost Prefix Rule). An index on `(tenant_id, status)` will optimize queries filtering on `tenant_id` or `(tenant_id, status)`, but *will not* optimize queries filtering on `status` alone.

### 2. EXPLAIN Query Plan
To optimize slow queries, run `EXPLAIN <query>` in your DB console. Key columns to inspect:
* `type`: Look for `const`, `ref`, or `range`. If it says `ALL`, a full table scan is occurring. An index is needed.
* `key`: Indicates the name of the index MySQL decided to use.
* `rows`: The estimate of records MySQL must scan to return results.

---

## 🚫 Debugging the N+1 Query Problem

The N+1 query problem occurs when an Object-Relational Mapper (ORM) executes N additional database queries to fetch related entities for a list of N primary records.

### 🔍 Real-World Example (ConnectWise Context)
Suppose we fetch a list of 10 ITBoost organizations and their matching integrations.
* **ORM Code (Sequential):**
  ```javascript
  const organizations = await Organization.find(); // 1 query
  for (const org of organizations) {
    org.integrations = await Integration.find({ organizationId: org.id }); // N queries
  }
  ```
  *Total Queries:* 1 + 10 = 11 queries. On production with 1,000 organizations, this executes 1,001 queries, overloading connection pools.

### 🛠️ Solution: Eager Loading (Join Fetching)
Instead of lazy loop querying, execute a SQL join or load relations using SQL `IN`:
* **Using SQL Join:**
  ```sql
  SELECT * FROM organizations o LEFT JOIN integrations i ON o.id = i.organization_id;
  ```
* **ORM Code (Eager Loading):**
  ```typescript
  // TypeORM / Sequelize
  const organizations = await Organization.find({ relations: ["integrations"] });
  ```
  *Total Queries:* 1 single optimized join query.

---

## 🔒 ACID Properties Deep Dive

A transaction is a single unit of work. To maintain database consistency, it must satisfy **ACID**:

1. **Atomicity:** "All or nothing". If one query in a transaction fails, the entire transaction is rolled back (e.g. money debited must be credited, or both fail).
2. **Consistency:** Moves the database from one valid state to another, maintaining constraints and foreign keys.
3. **Isolation:** Determines how transaction integrity is visible to other concurrent transactions.
   - *Read Committed:* Prevents dirty reads (reading uncommitted data).
   - *Repeatable Read (MySQL Default):* Ensures that if you read a row twice within a transaction, the values are identical.
   - *Serializable:* The strictest level, locking rows to run transactions sequentially.
4. **Durability:** Once a transaction commits, it remains saved even in the event of a system crash. MySQL achieves this by writing to a write-ahead log (Redo Log) before updating database pages.
