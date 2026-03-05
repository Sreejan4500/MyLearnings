# ClickHouse: Why It Can, Cannot, and What to Expect

A beginner-friendly guide to the **reasons** behind ClickHouse’s strengths, limitations, and design choices. Use this as a reference when learning or choosing ClickHouse.

---

## ClickHouse Can … (Why these strengths exist)

### 1. Store trillions of rows (petabytes)

**Why:** ClickHouse is built as a **columnar** database. Instead of storing rows one after another (row store), it stores each **column** separately. That gives:

- **Efficient compression** – similar values in a column (e.g. dates, categories) compress very well.
- **Read only what you need** – a query that uses 3 columns reads only those 3 columns from disk, not entire rows.
- **Batch processing** – data is written and read in large chunks (parts), which is ideal for analytics at scale.

So the “why” is: **columnar storage + compression + reading only needed columns** make it practical to store and query trillions of rows and petabytes of data.

---

### 2. Organise data in tables with hundreds of columns

**Why:** In a columnar layout, adding more columns mostly means adding more column files. Query cost depends on **how many columns you actually use** in the query, not how many exist in the table. So:

- Wide tables (hundreds of columns) are natural for analytics (many metrics and dimensions).
- You avoid the “one wide row” problem of row stores, where reading a few columns still touches the whole row.

The “why”: **columnar storage** makes wide tables cheap as long as each query touches a small subset of columns.

---

### 3. Retrieve results in milliseconds

**Why:** Several design choices add up:

- **Columnar storage** – read only the columns in your query.
- **Strong compression** – less I/O and better cache usage.
- **No general-purpose transaction engine** – no row-level locking or undo/redo; reads are simple scans over immutable data.
- **Data often partitioned and sorted** – e.g. by date; queries with time filters can skip whole chunks.
- **Vectorized execution** – CPU works on batches of values (SIMD), not one value at a time.

So: **fewer bytes read, less locking, and vectorized execution** explain why analytical queries can return in milliseconds.

---

### 4. Compress the data efficiently and store it

**Why:** Columnar layout is a perfect fit for compression:

- Values in one column are **similar** (same type, often similar values), so compression algorithms (e.g. LZ4, Zstd) work very well.
- Encodings like **delta encoding** (store differences) and **dictionary encoding** (store IDs for repeated values) are applied per column.

Result: **10x or much higher compression** is common, so you can keep more data on the same disk and speed up I/O.

---

## ClickHouse Cannot … (Why these limitations exist)

### 1. Stored procedures

**Why:** ClickHouse is an **analytical** database, not an OLTP (transactional) one. The design focuses on:

- Fast **ad-hoc** and **batch** queries over huge datasets.
- Simple execution model: you send a query, it runs, returns a result. No server-side procedural language, no persistent procedures.

Stored procedures would require a richer execution environment, session state, and control flow – which would complicate the engine and conflict with its “do one thing well” philosophy. So they are **out of scope by design**.

---

### 2. Full-fledged triggers

**Why:** Triggers imply **row-level or event-level** logic on every insert/update/delete, with ordering and consistency guarantees. ClickHouse is built for **bulk inserts** and **append-mostly** workloads:

- Data is written in **parts** (batches), not row-by-row.
- There is no fine-grained “after each row” execution model.

Adding real triggers would require a different write path and consistency model. **Materialized views** are the intended way to react to new data (they run when parts are merged/inserted), not row-level triggers.

---

### 3. Joins are slow

**Why:** ClickHouse is optimized for **single-table (or few-table) scans** and aggregations:

- **Joins** need to match rows between two (often large) datasets. That can mean hash tables in memory or heavy I/O.
- The primary use case is **denormalized** data: one big table with many columns, minimal joining. Joins are supported but not the main design target.

So joins are “slow” **relative to** its blazingly fast single-table scans; the engine is tuned for the latter, not for complex multi-table joins.

---

### 4. Eventually consistent

**Why:** ClickHouse favours **throughput and simplicity** over strong consistency:

- Inserts are applied to **replicas** asynchronously (replication is async).
- There is no distributed transaction protocol (e.g. two-phase commit) across nodes for every write.

So replicas can be **briefly behind** the primary. For analytics, “read your own writes after a short delay” is usually acceptable; for strict “read-after-write” consistency everywhere, you’d look at other systems.

---

### 5. Can handle up to a few hundred queries per second

**Why:** Each query is **CPU and I/O intensive** (scanning and aggregating large amounts of data). The design prioritises:

- **Throughput per query** (millions of rows per second per node).
- **Batch / analytical** usage: few concurrent users, heavy queries.

It is **not** tuned like an OLTP database for millions of small, simple queries per second. So the “few hundreds of QPS” reflects that it’s an **analytical** engine, not a high-QPS OLTP engine.

---

## But … (Workload and design expectations)

These points describe the **kind of workload** ClickHouse is built for and **what to expect** when using it.

### 1. More writes than reads (write-heavy ingestion)

**Why it’s mentioned:** In practice, you often **ingest** a lot (logs, events, metrics) and **query** less frequently. ClickHouse is built for that:

- Bulk inserts in large parts.
- Fewer, heavier read queries.

So “more writes than reads” is the **typical pattern** it’s optimized for, not a strict requirement. It can still serve read-heavy dashboards if the total QPS stays in the “few hundred” range.

---

### 2. Data is rarely modified (updated/deleted)

**Why:** The storage model is **append-oriented**:

- Data is written into **immutable parts**. Updates/deletes are implemented as **mutations** (rewriting parts or marking rows), which are heavier than appends.
- Fast path is **insert and merge**, not in-place update.

So the design assumes **append-mostly, rarely changing** data. Heavy update/delete workloads are not its strength.

---

### 3. Tables have a large number of columns

**Why:** This is the **ideal schema shape** for ClickHouse:

- Columnar storage makes many columns cheap; each column is a separate file.
- Analytical use cases often need many dimensions and metrics in one table (wide table).

So “tables with a large number of columns” is both **supported** and **recommended** when it matches your analytics model.

---

### 4. Data is filtered or aggregated while querying

**Why:** ClickHouse is built exactly for this:

- **Filtering** – `WHERE` with partition and sorting keys lets it skip entire parts and ranges.
- **Aggregation** – `GROUP BY`, `SUM`, `COUNT`, etc. are highly optimized (hash tables, parallel execution).

So the **typical access pattern** is “filter + aggregate over large datasets”, not “fetch one row by key”. The engine is tuned for this pattern.

---

### 5. No transactions (no ACID transactions)

**Why:** Full ACID (atomicity, consistency, isolation, durability) for arbitrary multi-statement transactions would require:

- Locking, undo logs, and a more complex write path.
- Different performance and implementation trade-offs.

ClickHouse prioritises **throughput and simplicity**:

- Inserts are atomic at the **part** (batch) level.
- There are no multi-statement transactions with rollback; you don’t get “BEGIN; …; COMMIT/ROLLBACK” like in PostgreSQL or MySQL.

So “no transactions” means **no classical OLTP-style transactions**; you get part-level atomicity and strong single-insert semantics, not full ACID across multiple operations.

---

## Summary table

| Topic | One-line “why” |
|-------|----------------|
| **Can: Store trillions of rows** | Columnar + compression + reading only needed columns. |
| **Can: Hundreds of columns** | Columnar storage makes wide tables cheap. |
| **Can: Millisecond queries** | Fewer columns read, no row locking, vectorized execution. |
| **Can: Efficient compression** | Similar values per column compress very well. |
| **Cannot: Stored procedures** | Analytical engine; no server-side procedural language by design. |
| **Cannot: Full triggers** | Bulk/append model; materialized views instead of row triggers. |
| **Cannot: Fast joins** | Optimized for single-table scans; joins are secondary. |
| **Cannot: Strong consistency** | Async replication; prioritises throughput. |
| **Cannot: Millions of QPS** | Built for heavy analytical queries, not OLTP. |
| **But: More writes than reads** | Typical workload it’s optimized for. |
| **But: Rarely modified data** | Append-oriented; mutations are costlier. |
| **But: Many columns** | Ideal schema for columnar storage. |
| **But: Filter/aggregate** | Core use case; WHERE + GROUP BY are first-class. |
| **But: No transactions** | Part-level atomicity only; no OLTP-style ACID. |

---

## How to use this doc

- Keep this file in your repo (e.g. `docs/` or project root) and link to it from your main README.
- When someone asks “why can’t ClickHouse do X?” or “why is it so good at Y?”, use the sections above.
- Revisit the “Cannot” and “But” sections when designing schemas and choosing ClickHouse vs another database.

Good luck with your ClickHouse learning.
