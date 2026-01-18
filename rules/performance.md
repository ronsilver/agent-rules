---
trigger: glob
globs: ["*.py", "*.go", "*.js", "*.ts", "*.java"]
---

# Performance Best Practices

## Core Principles

1. **Measure First** - No optimization without profiling
2. **Algorithmic Complexity** - O(n) beats O(n²) always
3. **Database First** - Usually the bottleneck
4. **80/20 Rule** - 20% of code causes 80% of slowness

## Profiling Tools

| Language | CPU Profiler | Memory Profiler |
|----------|-------------|-----------------|
| Go | `go tool pprof` | `go tool pprof -alloc_space` |
| Python | `cProfile`, `py-spy` | `memory_profiler`, `tracemalloc` |
| Node.js | `--prof`, `clinic.js` | `--inspect` + Chrome DevTools |
| General | `perf`, `flamegraph` | `valgrind`, `heaptrack` |

```bash
# Go profiling
go test -cpuprofile=cpu.prof -memprofile=mem.prof -bench .
go tool pprof -http=:8080 cpu.prof

# Python profiling
python -m cProfile -o profile.prof app.py
python -m pstats profile.prof
```

## Algorithmic Complexity - MANDATORY

### Common Operations

| Data Structure | Access | Search | Insert | Delete |
|---------------|--------|--------|--------|--------|
| Array/List | O(1) | O(n) | O(n) | O(n) |
| HashMap/Dict | O(1) | O(1) | O(1) | O(1) |
| Set | - | O(1) | O(1) | O(1) |
| Binary Search | - | O(log n) | - | - |
| Sorted Array | O(1) | O(log n) | O(n) | O(n) |

### Anti-Pattern: O(n²) Loops

```python
# ❌ Bad - O(n*m) = O(n²) for similar sizes
def find_common(list_a, list_b):
    result = []
    for item in list_a:           # O(n)
        if item in list_b:        # O(m) - list search!
            result.append(item)
    return result                  # Total: O(n*m)

# ✅ Good - O(n+m) ≈ O(n)
def find_common(list_a, list_b):
    set_b = set(list_b)           # O(m)
    return [item for item in list_a if item in set_b]  # O(n)
```

```python
# ❌ Bad - Nested loops with repeated work
for order in orders:
    for item in order.items:
        product = db.query(Product).get(item.product_id)  # N*M queries!

# ✅ Good - Batch fetch
product_ids = {item.product_id for order in orders for item in order.items}
products = {p.id: p for p in db.query(Product).filter(Product.id.in_(product_ids))}
for order in orders:
    for item in order.items:
        product = products[item.product_id]  # O(1) lookup
```

## Database Performance

### N+1 Query Problem - FORBIDDEN

The agent **MUST** detect and reject N+1 queries:

```python
# ❌ N+1 Problem - 1 + N queries
users = User.query.all()           # Query 1: SELECT * FROM users
for user in users:
    print(user.profile.bio)        # Query 2-N: SELECT * FROM profiles WHERE user_id = ?

# ✅ Eager Loading - 1 or 2 queries
# Option 1: JOIN (1 query)
users = User.query.options(joinedload(User.profile)).all()

# Option 2: Subquery (2 queries)
users = User.query.options(subqueryload(User.profile)).all()
```

```go
// Go with GORM
// ❌ N+1
var users []User
db.Find(&users)
for _, user := range users {
    db.Model(&user).Association("Profile").Find(&user.Profile)  // N queries!
}

// ✅ Preload
var users []User
db.Preload("Profile").Find(&users)  // 2 queries total
```

### Indexing Strategy

| Query Pattern | Index Needed |
|--------------|--------------|
| `WHERE email = ?` | Index on `email` |
| `WHERE user_id = ? AND status = ?` | Composite index `(user_id, status)` |
| `WHERE created_at > ?` | Index on `created_at` |
| `ORDER BY created_at DESC` | Index on `created_at` |
| `JOIN users ON orders.user_id = users.id` | Index on `orders.user_id` |

```sql
-- Check for missing indexes (PostgreSQL)
SELECT schemaname, tablename, attname, null_frac, n_distinct
FROM pg_stats
WHERE tablename = 'orders';

-- Create index
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);

-- Composite index (order matters!)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

### Query Optimization

```sql
-- ❌ Bad - SELECT *
SELECT * FROM users WHERE id = 1;

-- ✅ Good - Only needed columns
SELECT id, name, email FROM users WHERE id = 1;

-- ❌ Bad - LIKE with leading wildcard (can't use index)
SELECT * FROM users WHERE email LIKE '%@gmail.com';

-- ✅ Good - Trailing wildcard (can use index)
SELECT * FROM users WHERE email LIKE 'john%';

-- Use EXPLAIN to analyze
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;
```

### Connection Pooling

```python
# SQLAlchemy connection pool
engine = create_engine(
    DATABASE_URL,
    pool_size=10,        # Maintain 10 connections
    max_overflow=20,     # Allow 20 more under load
    pool_timeout=30,     # Wait 30s for connection
    pool_recycle=1800,   # Recycle connections after 30min
)
```

## Caching

### Cache-Aside Pattern

```python
def get_user(user_id: int) -> User:
    # 1. Check cache
    cache_key = f"user:{user_id}"
    cached = redis.get(cache_key)
    if cached:
        return User.parse_raw(cached)

    # 2. Cache miss - fetch from DB
    user = db.query(User).get(user_id)
    if not user:
        return None

    # 3. Populate cache with TTL
    redis.setex(cache_key, 300, user.json())  # 5 min TTL

    return user

def update_user(user_id: int, data: dict):
    # Update DB
    db.query(User).filter_by(id=user_id).update(data)

    # Invalidate cache
    redis.delete(f"user:{user_id}")
```

### Cache Key Design

```python
# Include version for cache invalidation
CACHE_VERSION = "v1"

def cache_key(entity: str, id: int) -> str:
    return f"{CACHE_VERSION}:{entity}:{id}"

# user:v1:123
# order:v1:456
```

### TTL Guidelines

| Data Type | TTL | Reason |
|-----------|-----|--------|
| Static config | 24h | Rarely changes |
| User profiles | 5-15min | Balance freshness |
| Search results | 1-5min | Acceptable staleness |
| Session data | 1h | Security |
| Real-time data | 0 (no cache) | Must be fresh |

## Memory Management

### Avoid Memory Leaks

```python
# ❌ Bad - Unbounded cache
cache = {}
def get_data(key):
    if key not in cache:
        cache[key] = expensive_computation(key)  # Grows forever!
    return cache[key]

# ✅ Good - LRU cache with max size
from functools import lru_cache

@lru_cache(maxsize=1000)
def get_data(key):
    return expensive_computation(key)
```

```go
// Go - Use sync.Pool for frequently allocated objects
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process() {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()
    // Use buf...
}
```

### Streaming Large Data

```python
# ❌ Bad - Load entire file into memory
def process_file(path):
    with open(path) as f:
        data = f.read()  # 10GB in memory!
    for line in data.split('\n'):
        process(line)

# ✅ Good - Stream line by line
def process_file(path):
    with open(path) as f:
        for line in f:  # One line at a time
            process(line)
```

## Frontend Performance (Core Web Vitals)

| Metric | Target | Measures |
|--------|--------|----------|
| **LCP** | < 2.5s | Largest Contentful Paint |
| **INP** | < 200ms | Interaction to Next Paint |
| **CLS** | < 0.1 | Cumulative Layout Shift |

**Optimization Techniques:**
- Image optimization: WebP/AVIF, lazy loading, responsive images
- Bundle splitting: Dynamic imports, tree shaking
- CDN: Static assets, edge caching
- Compression: Brotli > gzip
- Preloading: Critical resources, fonts

## Performance Checklist

- [ ] Profiled before optimizing
- [ ] No O(n²) algorithms where O(n) possible
- [ ] N+1 queries eliminated
- [ ] Database indexes on filtered/joined columns
- [ ] Connection pooling configured
- [ ] Caching with appropriate TTLs
- [ ] No memory leaks (bounded caches)
- [ ] Large data streamed, not loaded entirely
- [ ] Frontend meets Core Web Vitals
