# Hash Join vs Merge Join in ClickHouse

This document explains two fundamental join algorithms used in ClickHouse and other database systems: **Hash Join** and **Merge Join**. Understanding their working mechanisms, advantages, and limitations helps you write efficient queries and reason about join performance.

---

## Hash Join

### Purpose

Hash join is a method used to join two large datasets efficiently, especially when the datasets are too big to fit into memory entirely. It is well-suited for **equi-joins** (joins using the `=` operator), which are common in OLAP workloads.

### Working Mechanism

1. **Build Phase**  
   ClickHouse scans one of the datasets (typically the smaller one) and builds a **hash table** based on the join key(s). This hash table stores rows indexed by the join key.

2. **Probe Phase**  
   The second dataset is then scanned. For each row:
   - ClickHouse computes the hash of its join key.
   - It checks whether that key exists in the hash table built in the first phase.
   - If a match is found, rows from both datasets are combined according to the join type (inner, left, etc.).

### Advantages

- **Speed**: Hash joins are generally faster than merge joins for many workloads because they avoid sorting and can perform lookups in **O(1)** time.
- **Suitability**: Ideal for equi-joins, which are the norm in analytical queries.

### Limitations

- **Memory usage**: Hash joins can be memory-intensive because the hash table must fit in RAM. ClickHouse uses optimizations (e.g., partitioning) to handle large datasets.
- **Join type restrictions**: Hash joins work best with equi-joins; other join types may require different algorithms.

### ClickHouse Optimization

ClickHouse automatically chooses hash join when appropriate. You can often improve performance by considering data distribution and join type when writing queries.

---

## Merge Join

### Purpose

Merge join is used to efficiently join two **sorted** datasets, especially when the data is already sorted or can be sorted with minimal overhead. It is common in data warehousing and analytical systems.

### Working Mechanism

1. **Prerequisite**: Both datasets must be **sorted on the join key(s)**. If not, they must be sorted first, which adds cost.

2. **Process**:
   - Initialize pointers (or iterators) at the start of both datasets.
   - Compare the current rows’ join keys.
   - If the keys match, output the joined row and advance both pointers.
   - If one key is smaller, advance the pointer of the dataset with the smaller key.
   - Repeat until one dataset is exhausted.

### Advantages

- **Efficiency for sorted data**: When data is already sorted, merge join is very fast and often faster than hash join, with no hashing or large hash tables.
- **Memory usage**: Typically uses **less memory** than hash join, as it only needs to read and compare sorted streams.

### Limitations

- **Sorting requirement**: Both sides must be sorted on the join key; if not, sorting overhead can be significant.
- **Join types**: Best suited for inner and outer equi-joins; less effective for non-equi joins.

### Usage in ClickHouse

ClickHouse can use merge join when data is sorted or when explicitly instructed. It is not the default for all join operations. It is particularly useful in analytical queries where large, sorted datasets are common.

---

## Typical Use Cases for Merge Join

Merge join is widely used in databases and data processing systems where joining sorted data is efficient:

| Domain | Use Case |
|--------|----------|
| **Data warehousing & OLAP** | Large tables pre-sorted on key columns enable fast merge joins in analytical queries. |
| **Relational databases** | Oracle, SQL Server, PostgreSQL use merge join when data is sorted or when sorting is efficient for large tables. |
| **ETL & data integration** | Sorted datasets from different sources are merged efficiently in Extract, Transform, Load workflows. |
| **Index-based joining** | Indexes on join columns produce ordered data, making merge join a natural fit. |
| **Distributed systems** | In Spark, Hadoop (MapReduce), etc., merge join is used when data is partitioned and sorted across nodes, reducing shuffle. |
| **Big data & in-memory DBs** | Systems like ClickHouse use merge join to process sorted data streams efficiently. |

---

## Summary Comparison

| Aspect | Hash Join | Merge Join |
|--------|-----------|------------|
| **Sorting** | Not required | Both sides must be sorted on join key |
| **Lookup** | O(1) via hash table | Linear scan with two pointers |
| **Memory** | Higher (hash table in RAM) | Lower (streaming over sorted data) |
| **Best for** | Equi-joins, unsorted data | Equi-joins when data is already sorted |
| **Default in ClickHouse** | Often chosen automatically | Used when sorted or when explicitly suitable |

---

## Practical Takeaways

- Prefer **hash join** when data is not sorted and you need fast equi-joins; be mindful of memory.
- Prefer **merge join** when tables are already sorted (e.g., by table order or indexes) to save memory and avoid extra sorting.
- ClickHouse selects the join algorithm based on statistics and query shape; understanding both helps you design schemas and queries for better performance.
