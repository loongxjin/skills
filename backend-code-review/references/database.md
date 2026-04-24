# Database & Storage Review Checklist

## Checklist

### 3.1 Indexing & Queries

- [ ] All WHERE / JOIN / ORDER BY columns are evaluated for indexing
- [ ] Composite indexes follow the leftmost prefix rule; high-selectivity columns first
- [ ] Avoid single-column indexes on low-cardinality columns (e.g. gender)
- [ ] Single table index count is kept within 5-6
- [ ] No `SELECT *`; only required columns are listed
- [ ] No prefix wildcard `LIKE '%keyword%'` (use full-text index / Elasticsearch instead)
- [ ] Deep pagination uses cursor-based or ID-based paging (avoid `LIMIT 100000,10`)
- [ ] No N+1 queries (looping one-by-one); use JOIN or batch IN queries
- [ ] Parameter types match column types to avoid implicit conversion disabling indexes
- [ ] No `ORDER BY RAND()`

### 3.2 Transactions

- [ ] Transactions are short; lock hold time is minimized
- [ ] **No external I/O inside transactions** (RPC, HTTP requests are prohibited)
- [ ] Transaction scope is as small as possible, wrapping only operations that need consistency
- [ ] Long transactions are split; use state machine + compensation for eventual consistency

### 3.3 Data Growth

- [ ] Data volume and growth rate are assessed for each table
- [ ] `COUNT(*)` on large tables uses stat tables or cache instead
- [ ] Bulk `DELETE` is executed in batches
- [ ] VARCHAR lengths are set appropriately
- [ ] JOINs on large tables consider denormalized fields or wide tables

### 3.4 Data Consistency

- [ ] Monetary operations use strong consistency within a single transaction
- [ ] Cross-service operations use message queues / state machine / compensation for eventual consistency
- [ ] Reconciliation mechanism exists as a last resort (scheduled job comparing upstream and downstream data)

## Anti-pattern Example

```
// 🔴 RPC call inside a transaction — dangerous! Network call can take seconds
tx := db.Begin()
tx.Create(&order)
paymentClient.Charge(order)  // network call holds DB lock
tx.Commit()

// ✅ Correct: complete local transaction first, then call external service
```
