
# ALTER in SQL: SQL Server vs ClickHouse

## 1. Does MS SQL Server Have `ALTER`?
Yes. In Microsoft SQL Server, the `ALTER` statement exists and is commonly used to modify the **structure** of database objects such as tables, columns, and constraints.

Example:

```sql
ALTER TABLE Employees
ADD Age INT;
```

Common uses of `ALTER` in SQL Server:
- Add a column
- Modify a column
- Change NULL / NOT NULL
- Add or remove constraints
- Drop columns

Example operations:

```sql
-- Add column
ALTER TABLE Employees
ADD Age INT;

-- Modify column type
ALTER TABLE Employees
ALTER COLUMN Name VARCHAR(200);

-- Allow NULL
ALTER TABLE Employees
ALTER COLUMN Age INT NULL;

-- Drop column
ALTER TABLE Employees
DROP COLUMN Age;
```

---

# 2. Is `ALTER` a DML Command?

No.

`ALTER` is part of **DDL (Data Definition Language)**.

## SQL Command Categories

| Category | Purpose | Examples |
|---|---|---|
| DDL | Defines or changes database structure | CREATE, ALTER, DROP, TRUNCATE |
| DML | Manipulates table data | INSERT, UPDATE, DELETE, SELECT |
| DCL | Controls permissions | GRANT, REVOKE |
| TCL | Manages transactions | COMMIT, ROLLBACK |

Why `ALTER` is DDL:

It modifies **table schema**, not the data inside the table.

Example:

```sql
ALTER TABLE Employees
ADD Salary INT;
```

---

# 3. What About ClickHouse?

ClickHouse behaves differently.

Some `ALTER` operations in ClickHouse actually **modify table data**.

Example:

```sql
ALTER TABLE trades
UPDATE price = price * 1.1
WHERE symbol = 'AAPL';
```

```sql
ALTER TABLE trades
DELETE WHERE symbol = 'AAPL';
```

These operations update or delete rows.

ClickHouse calls these operations **mutations**.

---

# 4. Why ClickHouse Uses `ALTER UPDATE` Instead of `UPDATE`

ClickHouse is a **column-oriented analytical database**.

Unlike traditional databases, it stores data in **immutable data parts**.

Because of this:

- Rows cannot be updated directly.
- Instead, ClickHouse rewrites entire data parts.
- This rewriting happens **in the background**.

So an operation like:

```sql
ALTER TABLE trades
UPDATE price = 100
WHERE id = 1;
```

actually performs a **mutation**, rewriting affected data blocks.

---

# 5. SQL Server vs ClickHouse: Core Architecture

| Feature | SQL Server | ClickHouse |
|---|---|---|
| Database type | OLTP (Transactional) | OLAP (Analytical) |
| Storage | Row-based | Column-based |
| Updates | Frequent and efficient | Expensive |
| Transactions | Full ACID | Limited |
| Data modification | Direct row update | Background mutation |

---

# 6. How `ALTER` is Used in Each Database

## SQL Server

`ALTER` is used strictly for **schema changes**.

Example:

```sql
ALTER TABLE Employees
ADD Age INT;
```

Data updates use different commands:

```sql
UPDATE Employees
SET Age = 30
WHERE Id = 1;
```

Clear separation:

| Purpose | Command |
|---|---|
Schema modification | ALTER |
Data modification | UPDATE / DELETE |

---

## ClickHouse

`ALTER` is used for both:

- Schema changes
- Data mutations

Examples:

```sql
ALTER TABLE trades ADD COLUMN price Float64;
```

```sql
ALTER TABLE trades UPDATE price = price * 1.1 WHERE symbol = 'AAPL';
```

```sql
ALTER TABLE trades DELETE WHERE symbol = 'AAPL';
```

---

# 7. Execution Behavior

| Feature | SQL Server | ClickHouse |
|---|---|---|
| Update execution | Immediate | Background mutation |
| Locking | Row/page locks | No row-level locks |
| Data rewrite | Only affected rows | Entire data parts |
| Transactions | Strong | Limited |

---

# 8. Performance Implications

## SQL Server

Best suited for:
- Banking systems
- Order processing
- Trading platforms
- Applications with frequent updates

## ClickHouse

Best suited for:
- Analytics
- Log processing
- Event data
- Large aggregations
- Data warehouses

Frequent updates in ClickHouse are expensive because the system must **rewrite data parts**.

---

# 9. Key Takeaway

| Topic | SQL Server | ClickHouse |
|---|---|---|
`ALTER` purpose | Schema changes | Schema + mutations |
Data updates | UPDATE | ALTER UPDATE |
Deletes | DELETE | ALTER DELETE |
Execution | Immediate | Background |
Architecture | Row store | Column store |

---

# Final Concept

SQL Server behaves like a **transactional system optimized for frequent row changes**.

ClickHouse behaves like an **analytical engine optimized for massive reads and aggregations**, where updates are implemented through **mutations using ALTER**.
