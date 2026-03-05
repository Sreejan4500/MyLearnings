# PostgreSQL vs ClickHouse for Tick OHLC Storage & Backtesting

**Use case:** Store OHLC (Open, High, Low, Close) of **every tick** for all instruments on **NSE** (equity/derivatives) and **Delta Exchange** (crypto), then use this data for **backtesting** (indicator-based and time-based strategies).

**Schema (summary):** ~4 columns (OHLC) + symbol, instrument_token, master_symbol_id, time + 1–2 more → **~8–10 columns**, fixed schema, append-only.

**Primary query pattern:** Look up OHLC by **MasterSymbolId** and **time range** (e.g. 01-03-2026 to 06-03-2026). Backtesting will repeatedly run queries of the form: “give me all OHLC for this instrument in this date range.”

This document compares PostgreSQL and ClickHouse for this purpose and gives a clear recommendation.

---

## 1. Use Case in Short

| Aspect | Description |
|--------|-------------|
| **Data** | Tick-level OHLC for every instrument (NSE + Delta Exchange). |
| **Scale** | Very large: many instruments × many ticks per second × long history. |
| **Writes** | Continuous, append-only; high ingest rate. |
| **Reads** | Backtesting: **search by MasterSymbolId + time range** (e.g. 01-03-2026 to 06-03-2026); then time-range scans, aggregations, indicator-like calculations. |
| **Updates/Deletes** | Rare (corrections only). |
| **Columns** | Few, known: OHLC + symbol, instrument_token, master_symbol_id, time + 1–2. |

---

## 2. Why This Use Case Is Demanding

- **Volume:** NSE + Delta have thousands of instruments; each can produce many ticks per second. Over months/years you get **billions to trillions** of rows.
- **Ingest:** Sustained high write throughput (append-only).
- **Backtesting:** Reads are **bulk**, **time-ordered**, and **filtered by MasterSymbolId + time range**. Typical query: “OHLC for MasterSymbolId = X and time between 01-03-2026 and 06-03-2026.” You then need:
  - All ticks for that instrument in [t1, t2], or
  - Aggregations / rolling windows over that range.
- **Consistency:** No need for multi-row transactions; append consistency per batch is enough.

So you need: **high append throughput**, **efficient time-range scans**, **good compression**, and **scalability** to very large tables. OLTP features (transactions, point lookups, high random-read QPS) are secondary.

---

## 3. Comparison Overview

| Criterion | PostgreSQL | ClickHouse |
|-----------|------------|-----------|
| **Storage model** | Row-based (default); columnar possible via extensions | Native columnar |
| **Append throughput** | Good with partitioning/batching; not built for “firehose” | Built for very high append throughput |
| **Compression** | Moderate (row storage + TOAST) | Very strong (columnar compression) |
| **Time-range / bulk scans** | OK with partitioning + indexes; degrades at huge scale | Excellent; core strength |
| **Aggregations (SUM, AVG, windows)** | Good, but full row access can be costly | Excellent; only touched columns read |
| **Scale (billions/trillions of rows)** | Possible with partitioning and tuning; hardware cost grows | Designed for this scale |
| **Transactions (ACID)** | Full support | No multi-statement transactions; part-level atomicity |
| **Point lookups (e.g. by id)** | Excellent (B-tree) | Possible but not optimal |
| **Joins** | Strong | Supported but secondary; denormalized preferred |
| **Ecosystem / SQL** | Mature, rich (TimescaleDB, Citus, etc.) | SQL, good for analytics; different ecosystem |

---

## 4. PostgreSQL for This Use Case

### Strengths

- **Familiarity:** Widely used; many devs know it. Rich tooling and drivers.
- **ACID:** Full transactions if you ever need “insert ticks + update state” in one transaction.
- **Flexibility:** Complex queries, joins, triggers, stored procedures, multiple indexes.
- **Time-series extensions:** e.g. **TimescaleDB** (hypertables, compression, time-based partitioning) improve fit for tick data.
- **Point lookups:** If you need “get tick by id” or “last tick for symbol,” indexes work very well.

### Weaknesses for This Workload

- **Row storage:** Reading “time, open, high, low, close, symbol” still touches full rows (or large parts of them). For billions of rows, I/O and cache efficiency suffer.
- **Scale:** At hundreds of billions / trillions of ticks, keeping scans and vacuum under control is harder; hardware and tuning effort grow.
- **Compression:** Not as effective as columnar for numeric OHLC columns repeated over time.
- **Append throughput:** Can be good with unlogged tables and batching, but not designed for “log-style” firehose at the same level as ClickHouse.

### When PostgreSQL (or TimescaleDB) Might Still Be Reasonable

- You need strict ACID or complex multi-table transactions.
- Total tick volume is **moderate** (e.g. not “all symbols, full history”) and you want one familiar database for both tick storage and other app data.
- You rely heavily on point lookups or complex joins on this same data.

---

## 5. ClickHouse for This Use Case

### Strengths

- **Columnar storage:** Only columns used in a query (e.g. time, O, H, L, C, symbol, instrument_token) are read. Ideal for backtesting queries that don’t need the full row.
- **Compression:** OHLC and time columns compress very well (lots of similar, ordered values). Saves space and speeds up I/O.
- **Append-only:** Matches ClickHouse’s preferred write pattern; no need for updates/deletes.
- **Lookup by MasterSymbolId + time range:** This is the main read pattern. With **ordering key** `(master_symbol_id, timestamp)`, ClickHouse can efficiently return OHLC for one instrument in a date range (e.g. 01-03-2026 to 06-03-2026): it skips irrelevant symbol ranges and scans only the matching time span. Partitioning by time (e.g. month) further prunes data.
- **Aggregations:** SUM, AVG, window functions, etc. are fast; again, only needed columns are scanned.
- **Scale:** Designed for tables with trillions of rows and petabytes; fits “all instruments, full tick history.”
- **Throughput:** Can ingest millions of rows per second per node with batched inserts.

### Weaknesses (Less Relevant Here)

- No multi-statement transactions (not needed for append-only tick + backtesting).
- Joins are costlier (your backtesting can be mostly single-table or pre-joined).
- Point lookups are not the forte (backtesting is range-based).
- Eventually consistent replication (usually acceptable for historical tick data).

### Schema / Design Hints for ClickHouse

- **Primary query:** “OHLC where master_symbol_id = ? AND time BETWEEN ? AND ?” (e.g. 01-03-2026 to 06-03-2026). Design the table so this is the fastest path.
- **Ordering key:** Use `(master_symbol_id, timestamp)`. This matches your search pattern: filter by MasterSymbolId, then by time range. ClickHouse can skip to the right symbol and read only that time range.
- **Partition key:** e.g. `toYYYYMM(timestamp)` so time-range queries (like 01-03–06-03-2026) touch only the relevant month(s).
- **Columns:** Exactly what you said: OHLC, symbol, instrument_token, master_symbol_id, time + 1–2. Use appropriate types (e.g. Float32/64 for OHLC, LowCardinality for symbol if helpful).
- **Engine:** ReplacingMergeTree or MergeTree; no need for heavy update/delete support.

---

## 6. Backtesting Access Pattern

Typical backtesting pattern:

1. **Input:** Strategy, **MasterSymbolId**, and **time range** (e.g. 01-03-2026 to 06-03-2026).
2. **Load:** Query OHLC by MasterSymbolId + time range; get all ticks for that instrument in that range, in time order.
3. **Compute:** Indicators (e.g. SMA, RSI) and signals in application code or via SQL (window functions).
4. **Evaluate:** P&L, drawdown, etc.

- In **PostgreSQL:** Large time-range scan over a partitioned table; row storage means reading more data than necessary if you only need a few columns.
- In **ClickHouse:** Same query reads only the columns you need, in column order, with strong compression and partition pruning. Fits the engine’s design.

So for **“OHLC by MasterSymbolId + time range”** (e.g. 01-03-2026 to 06-03-2026) plus a few columns, ClickHouse is a better fit: the ordering key `(master_symbol_id, timestamp)` and time partitioning align directly with this pattern.

---

## 7. Final Recommendation

**Use ClickHouse** for storing tick OHLC for all NSE and Delta Exchange instruments and for backtesting.

**Reasons in short:**

1. **Scale:** “Every tick, all instruments” implies very large tables. ClickHouse is built for this; PostgreSQL can do it but with more effort and cost at that scale.
2. **Access pattern:** Backtesting is **search by MasterSymbolId + time range** (e.g. 01-03-2026 to 06-03-2026), then time-ordered scans over a few columns and aggregations. ClickHouse’s ordering key and partitioning are built for this.
3. **Writes:** Append-only, high throughput suits ClickHouse’s insert model.
4. **Schema:** Few, fixed columns (OHLC + metadata) are ideal for columnar compression and fast scans.
5. **No need for:** Multi-row transactions, high random-read QPS, or complex joins on the tick table. So ClickHouse’s limitations are not blockers.

**Use PostgreSQL (or TimescaleDB)** if:

- You want a single database for both tick data and other transactional app data and your tick volume is not “all symbols, full history,” or
- You have a strong requirement for ACID transactions across tick inserts and other tables, or
- Your team has no capacity to operate a second system and you accept higher storage and query cost at scale.

---

## 8. Summary Table

| Factor | Better fit |
|--------|------------|
| Tick volume (billions–trillions of rows) | ClickHouse |
| Append-only ingest | ClickHouse |
| Lookup by MasterSymbolId + time range (e.g. 01-03–06-03-2026) | ClickHouse |
| Few columns (OHLC + metadata) | ClickHouse |
| Compression & storage cost | ClickHouse |
| ACID transactions | PostgreSQL |
| Single DB for app + ticks (moderate scale) | PostgreSQL / TimescaleDB |
| Point lookups, complex joins on tick data | PostgreSQL |

**Bottom line:** For **storing OHLC of every tick for NSE + Delta Exchange and backtesting**, **ClickHouse is the better choice** by design. Use the earlier “[ClickHouse: Why It Can, Cannot, and What to Expect](clickhouse-whys-beginner-guide.md)” doc to align schema and queries with its strengths and limitations.
