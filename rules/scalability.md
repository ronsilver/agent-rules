---
trigger: glob
globs: ["*.py", "*.go", "*.js", "*.ts", "*.tf", "docker-compose*.yml", "*.yaml"]
---

# Scalability Best Practices

## Core Principles

| Principle | Description |
|-----------|-------------|
| **Stateless** | No local state between requests |
| **Horizontal** | Scale by adding instances, not bigger servers |
| **Async** | Offload heavy work to queues |
| **Cache** | Reduce database load |
| **Partition** | Distribute data across shards |

## Stateless Services - NON-NEGOTIABLE

Application services **MUST NOT** store state locally that persists between requests.

| State Type | ❌ Wrong | ✅ Correct |
|-----------|----------|-----------|
| Sessions | In-memory dict | Redis / Database |
| File uploads | Local disk | S3 / Blob Storage |
| Cache | Local memory | Redis / Memcached |
| Locks | File locks | Redis distributed locks |
| Scheduled jobs | In-process timers | External scheduler (cron, CloudWatch) |

```python
# ❌ Bad - State in memory (lost on restart/scale)
user_sessions = {}  # Global dict

@app.route("/login")
def login():
    user_sessions[user_id] = session_data
    return {"status": "ok"}

# ✅ Good - State in Redis
@app.route("/login")
def login():
    redis.setex(f"session:{user_id}", 3600, json.dumps(session_data))
    return {"status": "ok"}
```

## Database Scalability

### Connection Pooling - MANDATORY

Never create connections per request:

```python
# ❌ Bad - Connection per request
def get_user(user_id):
    conn = psycopg2.connect(DATABASE_URL)  # Expensive!
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    return cursor.fetchone()

# ✅ Good - Connection pool
from sqlalchemy import create_engine
engine = create_engine(DATABASE_URL, pool_size=10, max_overflow=20)

def get_user(user_id):
    with engine.connect() as conn:
        return conn.execute("SELECT * FROM users WHERE id = %s", (user_id,)).fetchone()
```

**Pool sizing:** `pool_size = (core_count * 2) + effective_spindle_count`

### Read Replicas

Split reads and writes for high-volume applications:

```python
# Configuration
PRIMARY_DB = "postgresql://primary:5432/db"
REPLICA_DB = "postgresql://replica:5432/db"

# Route queries
def get_user(user_id):
    # Read from replica
    with replica_engine.connect() as conn:
        return conn.execute(query, (user_id,))

def update_user(user_id, data):
    # Write to primary
    with primary_engine.connect() as conn:
        conn.execute(update_query, (data, user_id))
```

### Indexing Strategy

| Query Pattern | Index Type |
|--------------|------------|
| `WHERE column = ?` | B-tree (default) |
| `WHERE column IN (...)` | B-tree |
| `WHERE column LIKE 'prefix%'` | B-tree |
| `WHERE column @> '{...}'` (JSON) | GIN |
| Full-text search | GIN with tsvector |
| Geospatial | GiST / SP-GiST |

## Async Processing - MANDATORY for Heavy Tasks

Offload work that takes > 500ms:

```python
# ❌ Bad - Blocking request
@app.route("/orders", methods=["POST"])
def create_order():
    order = save_order(data)
    send_confirmation_email(order)      # 2s
    generate_invoice_pdf(order)         # 3s
    notify_warehouse(order)             # 1s
    return {"order_id": order.id}       # Total: 6s+ response time

# ✅ Good - Async processing
@app.route("/orders", methods=["POST"])
def create_order():
    order = save_order(data)

    # Queue background tasks
    send_confirmation_email.delay(order.id)
    generate_invoice_pdf.delay(order.id)
    notify_warehouse.delay(order.id)

    return {"order_id": order.id, "status": "processing"}  # ~100ms
```

**Queue Technologies:**
| Use Case | Technology |
|----------|------------|
| Simple tasks | Redis + Celery/RQ |
| High throughput | Kafka / AWS SQS |
| Workflows | Temporal / AWS Step Functions |

## Caching Strategy

### Cache-Aside Pattern
```python
def get_user(user_id):
    # 1. Check cache
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)

    # 2. Cache miss - fetch from DB
    user = db.query(User).get(user_id)

    # 3. Populate cache with TTL
    redis.setex(f"user:{user_id}", 300, json.dumps(user.to_dict()))

    return user
```

### Cache Invalidation
```python
def update_user(user_id, data):
    # Update database
    db.query(User).filter_by(id=user_id).update(data)

    # Invalidate cache
    redis.delete(f"user:{user_id}")
```

**TTL Guidelines:**
| Data Type | TTL | Reason |
|-----------|-----|--------|
| Static config | 24h | Rarely changes |
| User profiles | 5-15min | Balance freshness/load |
| Session data | 1h | Security |
| Search results | 1-5min | Acceptable staleness |

## Resilience Patterns

### Rate Limiting - MANDATORY for APIs

```python
from flask_limiter import Limiter

limiter = Limiter(app, key_func=get_remote_address)

@app.route("/api/users")
@limiter.limit("100/minute")  # Per IP
def list_users():
    return users

# Return 429 Too Many Requests when exceeded
```

### Circuit Breaker

Fail fast when dependencies are unhealthy:

```python
import circuitbreaker

@circuitbreaker.circuit(failure_threshold=5, recovery_timeout=30)
def call_payment_service(order):
    return requests.post(PAYMENT_URL, json=order.to_dict())

# After 5 failures, circuit opens
# Calls fail immediately for 30s (no waiting for timeout)
# After 30s, circuit allows one test request
```

### Retry with Exponential Backoff

```python
import backoff

@backoff.on_exception(
    backoff.expo,           # Exponential backoff
    requests.RequestException,
    max_tries=5,
    max_time=60,
    jitter=backoff.full_jitter  # Add randomness
)
def fetch_data(url):
    response = requests.get(url, timeout=5)
    response.raise_for_status()
    return response.json()

# Retry delays: ~1s, ~2s, ~4s, ~8s, ~16s (with jitter)
```

### Bulkhead Pattern

Isolate failures to prevent cascade:

```python
from concurrent.futures import ThreadPoolExecutor

# Separate pools for different dependencies
payment_pool = ThreadPoolExecutor(max_workers=10)
inventory_pool = ThreadPoolExecutor(max_workers=10)

# Payment service issues won't exhaust inventory threads
def process_order(order):
    payment_future = payment_pool.submit(charge_card, order)
    inventory_future = inventory_pool.submit(reserve_items, order)

    return payment_future.result(), inventory_future.result()
```

## Horizontal Scaling Checklist

- [ ] Services are stateless
- [ ] Sessions stored externally (Redis)
- [ ] Files stored in object storage (S3)
- [ ] Database connections pooled
- [ ] Heavy tasks queued (async)
- [ ] Caching implemented with TTL
- [ ] Rate limiting configured
- [ ] Circuit breakers on external calls
- [ ] Health checks implemented (`/health`, `/ready`)
- [ ] Graceful shutdown handling
