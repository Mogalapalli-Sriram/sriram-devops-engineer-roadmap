# System Design Practice - Senior DevOps Engineer

## HOW TO APPROACH SYSTEM DESIGN INTERVIEWS

### Framework (Use for Every Problem)

**1. Requirements Clarification (5 minutes)**
```
Functional Requirements:
- What does the system need to do?
- What are the core features?
- What are the APIs/interfaces?

Non-Functional Requirements:
- Scale: Users? Requests/sec? Data volume?
- Performance: Latency requirements?
- Availability: Uptime SLA?
- Consistency: Strong vs eventual?
- Security: Authentication? Authorization?
```

**2. Back-of-the-Envelope Estimation (5 minutes)**
```
- Daily Active Users (DAU)
- Requests per second (QPS)
- Storage requirements
- Bandwidth requirements
- Memory/cache requirements
```

**3. High-Level Design (10 minutes)**
```
- Draw major components (boxes)
- Show data flow (arrows)
- Identify databases, caches, queues
- Explain why you chose each component
```

**4. Deep Dive (15 minutes)**
```
- Detailed design of 2-3 critical components
- Database schema
- API contracts
- Caching strategy
- Scaling approach
```

**5. Trade-offs & Alternatives (5 minutes)**
```
- Discuss pros/cons of your choices
- Alternative approaches
- What would you do differently at 10x scale?
```

---

## SYSTEM DESIGN PROBLEM 1: URL SHORTENER (bit.ly)

### Requirements Clarification

**Interviewer:** "Design a URL shortening service like bit.ly."

**You:** "Let me clarify the requirements:

**Functional:**
1. Shorten long URL to short URL
2. Redirect short URL to original URL
3. Custom aliases (optional)?
4. Analytics (click tracking)?
5. Expiration for URLs?

**Non-Functional:**
1. Scale: 100M URLs created, 10B redirects/month?
2. Availability: 99.99% uptime?
3. Latency: <100ms for redirects?
4. Reads vs Writes: 100:1 ratio (mostly redirects)?

Let's assume:
- 100M new URLs/month = 40 writes/sec
- 10B redirects/month = 4,000 reads/sec
- URL stored for 5 years
- 99.99% availability required"

### Back-of-the-Envelope Calculations

```
Storage Estimation:
- 100M new URLs/month × 12 months × 5 years = 6B URLs
- Per URL: 500 bytes (original_url: 200B, short_url: 10B, metadata: 290B)
- Total: 6B × 500B = 3TB

Bandwidth:
- Writes: 40 URLs/sec × 500B = 20KB/sec (negligible)
- Reads: 4,000 redirects/sec × 500B = 2MB/sec

Cache:
- 80/20 rule: 20% URLs get 80% traffic
- Cache 20% of daily URLs = 20% × 3.3M = 660K URLs
- Memory: 660K × 500B = 330MB (easily fits in cache)
```

### High-Level Design

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       │ HTTPS
       │
┌──────▼───────────────────────────────────────┐
│           CloudFront (CDN)                   │
│  - Global edge locations                     │
│  - Cache redirects (TTL: 1 hour)            │
└──────┬───────────────────────────────────────┘
       │
       │
┌──────▼──────────────────────────────────────┐
│        API Gateway                          │
│  - Rate limiting (1000 req/sec per user)   │
│  - API keys / authentication               │
└──────┬──────────────────────────────────────┘
       │
       ├─────────────┬──────────────┐
       │             │              │
   ┌───▼────┐   ┌───▼────┐    ┌───▼────┐
   │Lambda  │   │Lambda  │    │Lambda  │
   │Shorten │   │Redirect│    │Analytics│
   └───┬────┘   └───┬────┘    └───┬────┘
       │            │              │
       │            │              │
   ┌───▼────────────▼──────────────▼─────┐
   │         DynamoDB                     │
   │  - Partition Key: short_url         │
   │  - Attributes: original_url, created│
   │  - Global Secondary Index: user_id  │
   └──────────────────────────────────────┘
```

### Detailed Component Design

**1. Short URL Generation**

**Approach 1: Hash-based (MD5/SHA256)**
```python
import hashlib

def generate_short_url(original_url):
    # MD5 hash of URL
    hash_value = hashlib.md5(original_url.encode()).hexdigest()
    
    # Take first 7 characters
    short_code = hash_value[:7]
    
    # Check collision in database
    if exists_in_db(short_code):
        # Append counter or use different hash
        short_code = hash_value[:6] + random_char()
    
    return f"bit.ly/{short_code}"

# Problem: Collisions possible, need to check DB
# Benefit: Same URL always generates same short code
```

**Approach 2: Auto-increment ID + Base62 Encoding (BETTER)**
```python
BASE62 = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

def encode_base62(num):
    if num == 0:
        return BASE62[0]
    
    result = []
    while num:
        result.append(BASE62[num % 62])
        num //= 62
    
    return ''.join(reversed(result))

# Example:
# ID 1 → "1"
# ID 62 → "10"
# ID 1000000 → "4c92"

# Benefits:
# - No collisions (guaranteed unique)
# - Short codes (7 chars = 62^7 = 3.5 trillion URLs)
# - Lexicographically sortable

# Implementation:
def create_short_url(original_url, user_id):
    # Get next ID from DynamoDB atomic counter
    url_id = get_next_id()
    
    short_code = encode_base62(url_id)
    
    # Store in DynamoDB
    dynamodb.put_item({
        'short_url': short_code,
        'original_url': original_url,
        'user_id': user_id,
        'created_at': int(time.time()),
        'clicks': 0
    })
    
    return f"bit.ly/{short_code}"
```

**2. DynamoDB Schema**
```python
# Table: URLs
{
    "short_url": "abc123",      # Partition Key
    "original_url": "https://...",
    "user_id": "user_789",
    "created_at": 1640000000,
    "expires_at": 1800000000,   # Optional
    "clicks": 1523,
    "last_accessed": 1650000000
}

# Global Secondary Index: user_id (for "my URLs" lookup)
GSI_UserUrls:
    Partition Key: user_id
    Sort Key: created_at
```

**3. Lambda Function: Redirect**
```python
import boto3
import json

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('URLs')

def lambda_handler(event, context):
    short_code = event['pathParameters']['short_code']
    
    # Query DynamoDB
    response = table.get_item(Key={'short_url': short_code})
    
    if 'Item' not in response:
        return {
            'statusCode': 404,
            'body': json.dumps({'error': 'URL not found'})
        }
    
    original_url = response['Item']['original_url']
    
    # Increment click counter (async to not block redirect)
    table.update_item(
        Key={'short_url': short_code},
        UpdateExpression='SET clicks = clicks + :inc, last_accessed = :time',
        ExpressionAttributeValues={
            ':inc': 1,
            ':time': int(time.time())
        }
    )
    
    # 301 Permanent Redirect (cacheable)
    return {
        'statusCode': 301,
        'headers': {
            'Location': original_url,
            'Cache-Control': 'public, max-age=3600'  # Cache for 1 hour
        }
    }
```

**4. Caching Strategy**

```
Client Request: bit.ly/abc123
       ↓
   CloudFront Edge
       ↓ [Cache Hit?]
   ┌───Yes → Return cached redirect (3ms)
   │
   └───No → API Gateway → Lambda → DynamoDB (50ms)
              ↓
         Cache result in CloudFront for 1 hour
```

**Cache Invalidation:**
- TTL: 1 hour (trade-off between freshness and performance)
- Manual invalidation if URL updated/deleted
- 99% hit rate after warm-up

### Scaling Considerations

**At 10x Scale (40,000 writes/sec, 40,000 reads/sec):**

**1. Write Path Bottleneck:**
- DynamoDB write capacity: Use on-demand mode (auto-scales)
- Or provisioned: 40K WCU (Write Capacity Units)
- Cost: ~$24K/month

**Optimization: Write Batching**
```python
# Instead of real-time counter updates, batch them:
Client → Kinesis Stream → Lambda (batches of 1000) → DynamoDB
# Reduces write load by 1000x
# Trade-off: Analytics delayed by 1-2 minutes
```

**2. Read Path:**
- DynamoDB: 40K RCU
- CloudFront cache hit rate: 95% → Only 2,000 req/sec hit DynamoDB
- Actual DynamoDB load: 2K RCU (very affordable)

**3. Hot URLs:**
```
Problem: Single popular URL gets 10K requests/sec
Solution:
- CloudFront distributes across edge locations
- DynamoDB handles 10K RCU easily (auto-partitions hot keys)
```

**4. Database Sharding (if needed at 100x scale):**
```
Partition by short_code prefix:
- Shard 0: short_codes starting with [0-9]
- Shard 1: short_codes starting with [a-m]
- Shard 2: short_codes starting with [n-z, A-Z]

Consistent hashing for even distribution
```

### Your Drishya Connection

"At Drishya, I implemented a similar serverless URL shortener for our internal campaign tracking:
- Used Lambda + API Gateway + DynamoDB (same stack)
- Handled 100K redirects/day
- Cost: $15/month (vs $500/month for third-party service)
- Sub-100ms p95 latency globally with CloudFront
- Built in 2 weeks as a side project, now used company-wide"

---

## SYSTEM DESIGN PROBLEM 2: REAL-TIME ANALYTICS PLATFORM

### Requirements

**Interviewer:** "Design a real-time analytics platform for an e-commerce site. Users should see live dashboards showing orders, revenue, traffic."

**Clarifying Questions:**
```
Functional:
- What metrics? (Orders/min, revenue, active users, top products)
- Real-time definition? (<5 seconds latency)
- Historical data? (Last 30 days)
- Custom queries or predefined dashboards?

Non-Functional:
- Scale: 10,000 orders/sec, 100,000 page views/sec
- Availability: 99.9%
- Data retention: 30 days (hot), 1 year (cold)
```

### Back-of-the-Envelope

```
Events:
- Page views: 100K/sec = 8.6B/day
- Orders: 10K/sec = 864M/day
- Total events: 110K/sec

Storage:
- Per event: 500 bytes
- Daily: 110K × 86400 × 500B = 4.7TB/day
- 30 days (hot): 142TB
- 1 year (cold): 1.7PB

Processing:
- Real-time aggregations: SUM, COUNT, AVG
- Time windows: 1-minute, 5-minute, 1-hour
- Dashboards: 1000 concurrent users
```

### High-Level Architecture

```
┌──────────────────────────────────────────────┐
│         Event Producers                      │
│  - Web App (page views)                      │
│  - Checkout Service (orders)                 │
│  - Payment Service (transactions)            │
└──────────┬───────────────────────────────────┘
           │
           │ REST API / SDK
           │
┌──────────▼───────────────────────────────────┐
│      API Gateway / Load Balancer             │
│  - Rate limiting                             │
│  - Authentication                            │
└──────────┬───────────────────────────────────┘
           │
           │
┌──────────▼───────────────────────────────────┐
│       Kafka / Kinesis Stream                 │
│  - Partitioned by user_id                    │
│  - Retention: 7 days                         │
│  - Throughput: 110K events/sec               │
└──────────┬───────────────────────────────────┘
           │
           ├────────────┬─────────────┐
           │            │             │
   ┌───────▼──────┐ ┌──▼──────┐  ┌──▼─────────┐
   │ Flink/Spark  │ │Lambda   │  │ Firehose   │
   │ Streaming    │ │(light   │  │(S3 backup) │
   │(Aggregation) │ │ETL)     │  │            │
   └───────┬──────┘ └──┬──────┘  └──┬─────────┘
           │           │            │
   ┌───────▼───────────▼────────────▼──────────┐
   │    Time-Series Database                   │
   │    (TimescaleDB / InfluxDB)               │
   │  - 1-min granularity (30 days)            │
   │  - Automatic downsampling                 │
   └───────┬───────────────────────────────────┘
           │
   ┌───────▼───────────────────────────────────┐
   │         Redis Cache                       │
   │  - Last 1 hour data (hot cache)           │
   │  - Dashboard queries                      │
   └───────┬───────────────────────────────────┘
           │
   ┌───────▼───────────────────────────────────┐
   │         Grafana Dashboards                │
   │  - Real-time charts                       │
   │  - Custom alerts                          │
   └───────────────────────────────────────────┘

              [Cold Storage]
                    │
   ┌────────────────▼──────────────────────────┐
   │      S3 + Athena (Historical)             │
   │  - Parquet format                         │
   │  - 1 year retention                       │
   │  - Ad-hoc SQL queries                     │
   └───────────────────────────────────────────┘
```

### Detailed Design

**1. Ingestion Layer (Kafka/Kinesis)**

```python
# Event Schema (Avro)
{
    "namespace": "com.ecommerce.events",
    "type": "record",
    "name": "OrderEvent",
    "fields": [
        {"name": "event_id", "type": "string"},
        {"name": "event_type", "type": "string"},  # "page_view", "order", "payment"
        {"name": "timestamp", "type": "long"},     # Unix timestamp
        {"name": "user_id", "type": "string"},
        {"name": "session_id", "type": "string"},
        {"name": "data", "type": {
            "type": "map",
            "values": "string"
        }}  # Event-specific data
    ]
}

# Producer (from application)
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['kafka1:9092', 'kafka2:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    compression_type='snappy',  # Reduce network bandwidth
    batch_size=16384,          # Batch for efficiency
    linger_ms=10               # Wait 10ms to batch
)

def track_event(event_type, user_id, data):
    event = {
        'event_id': str(uuid.uuid4()),
        'event_type': event_type,
        'timestamp': int(time.time() * 1000),
        'user_id': user_id,
        'data': data
    }
    
    # Partition by user_id (maintains ordering per user)
    producer.send('events', value=event, key=user_id.encode('utf-8'))

# Kafka Configuration:
# - Partitions: 50 (for parallelism)
# - Replication factor: 3 (for durability)
# - Retention: 7 days
```

**2. Stream Processing (Apache Flink)**

```python
# Flink Job: Real-time Aggregations

from pyflink.datastream import StreamExecutionEnvironment
from pyflink.table import StreamTableEnvironment

env = StreamExecutionEnvironment.get_execution_environment()
t_env = StreamTableEnvironment.create(env)

# Define Kafka source
t_env.execute_sql("""
    CREATE TABLE events (
        event_id STRING,
        event_type STRING,
        timestamp BIGINT,
        user_id STRING,
        data MAP<STRING, STRING>,
        event_time AS TO_TIMESTAMP(FROM_UNIXTIME(timestamp/1000)),
        WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
    ) WITH (
        'connector' = 'kafka',
        'topic' = 'events',
        'properties.bootstrap.servers' = 'kafka1:9092',
        'format' = 'json'
    )
""")

# Aggregation: Orders per minute
t_env.execute_sql("""
    CREATE TABLE orders_per_minute AS
    SELECT
        TUMBLE_START(event_time, INTERVAL '1' MINUTE) as window_start,
        COUNT(*) as order_count,
        SUM(CAST(data['amount'] AS DECIMAL)) as total_revenue
    FROM events
    WHERE event_type = 'order'
    GROUP BY TUMBLE(event_time, INTERVAL '1' MINUTE)
""")

# Write to TimescaleDB
t_env.execute_sql("""
    CREATE TABLE timescale_sink (
        window_start TIMESTAMP,
        order_count BIGINT,
        total_revenue DECIMAL
    ) WITH (
        'connector' = 'jdbc',
        'url' = 'jdbc:postgresql://timescale:5432/analytics',
        'table-name' = 'orders_per_minute'
    )
""")

t_env.execute_sql("""
    INSERT INTO timescale_sink
    SELECT * FROM orders_per_minute
""")
```

**3. Storage Layer (TimescaleDB)**

```sql
-- TimescaleDB hypertable (optimized for time-series)
CREATE TABLE orders_per_minute (
    window_start TIMESTAMPTZ NOT NULL,
    order_count BIGINT,
    total_revenue DECIMAL,
    PRIMARY KEY (window_start)
);

-- Convert to hypertable (automatic partitioning by time)
SELECT create_hypertable('orders_per_minute', 'window_start');

-- Automatic data retention (drop data older than 30 days)
SELECT add_retention_policy('orders_per_minute', INTERVAL '30 days');

-- Continuous aggregates (pre-compute hourly from minutely)
CREATE MATERIALIZED VIEW orders_per_hour
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', window_start) AS bucket,
    SUM(order_count) as total_orders,
    SUM(total_revenue) as total_revenue
FROM orders_per_minute
GROUP BY bucket;

-- Refresh policy (update every 10 minutes)
SELECT add_continuous_aggregate_policy('orders_per_hour',
    start_offset => INTERVAL '2 hours',
    end_offset => INTERVAL '10 minutes',
    schedule_interval => INTERVAL '10 minutes');
```

**4. Caching Layer (Redis)**

```python
import redis
import json

redis_client = redis.Redis(host='redis', port=6379, db=0)

def cache_latest_metrics():
    # Cache last 1 hour of data for dashboard queries
    query = """
        SELECT window_start, order_count, total_revenue
        FROM orders_per_minute
        WHERE window_start > NOW() - INTERVAL '1 hour'
        ORDER BY window_start DESC
    """
    
    results = execute_query(query)
    
    # Store in Redis with 5-minute expiry
    redis_client.setex(
        'dashboard:orders:1h',
        300,  # 5 minutes TTL
        json.dumps(results)
    )

# Dashboard API (serves from cache)
def get_dashboard_data():
    cached = redis_client.get('dashboard:orders:1h')
    
    if cached:
        return json.loads(cached)  # Cache hit (fast)
    
    # Cache miss: query TimescaleDB, update cache
    return cache_latest_metrics()
```

**5. Visualization (Grafana)**

```sql
-- Grafana query for dashboard
SELECT
    $__timeGroup(window_start, '1m') as time,
    AVG(order_count) as "Orders per Minute",
    AVG(total_revenue) as "Revenue per Minute"
FROM orders_per_minute
WHERE
    $__timeFilter(window_start)
GROUP BY time
ORDER BY time
```

### Scaling & Trade-offs

**Write Path Bottleneck (at 10x scale):**

Problem: 1.1M events/sec overwhelming Kafka

Solutions:
```
1. Increase Kafka partitions: 50 → 500
2. Add more Kafka brokers: 3 → 20
3. Use Kafka compression (snappy): 30% bandwidth reduction
4. Client-side batching: Send 100 events per API call

Cost: Kafka cluster ~$10K/month (AWS MSK)
```

**Read Path Optimization:**

```
Problem: 1000 concurrent dashboards querying TimescaleDB

Solution 1: Redis cache
- 90% queries served from cache
- 10ms response time (vs 200ms from DB)

Solution 2: Read replicas
- TimescaleDB with 3 read replicas
- Round-robin load balancing

Solution 3: Pre-aggregation
- Continuous aggregates (minutely → hourly → daily)
- Query hourly data for longer time ranges (much faster)
```

### Your Drishya Connection

"At Drishya, I built a real-time analytics pipeline for Oil & Gas IoT sensors:

**Architecture:**
```
IoT Devices → AWS IoT Core → Kinesis Data Streams → 
Lambda (aggregation) → TimescaleDB → Grafana
```

**Scale:**
- 50K sensor readings/second
- 10+ GB/day
- <2 second latency (sensor → dashboard)

**Challenges:**
- Lambda timeout: Batched 1000 events per invocation
- TimescaleDB performance: Used continuous aggregates
- Cost: $800/month (vs $5K quote from third-party vendor)

**Result:**
- Enabled predictive maintenance (ML on real-time data)
- Detected anomalies 10 minutes faster than before
- Reduced downtime by 25%"

---

## KEY DESIGN PATTERNS FOR DEVOPS INTERVIEWS

### 1. API Rate Limiting
```python
# Token Bucket Algorithm
class RateLimiter:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate  # tokens/second
        self.last_refill = time.time()
    
    def allow_request(self):
        self._refill()
        
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
    
    def _refill(self):
        now = time.time()
        elapsed = now - self.last_refill
        new_tokens = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + new_tokens)
        self.last_refill = now

# AWS API Gateway implementation
{
    "throttle": {
        "burstLimit": 5000,  # Bucket capacity
        "rateLimit": 2000    # Refill rate (requests/sec)
    }
}
```

### 2. Circuit Breaker Pattern
```python
from enum import Enum

class State(Enum):
    CLOSED = 1    # Normal operation
    OPEN = 2      # Failing, reject requests
    HALF_OPEN = 3 # Testing if recovered

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.state = State.CLOSED
        self.last_failure_time = None
    
    def call(self, func, *args, **kwargs):
        if self.state == State.OPEN:
            if time.time() - self.last_failure_time > self.timeout:
                self.state = State.HALF_OPEN  # Try again
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e
    
    def _on_success(self):
        self.failure_count = 0
        self.state = State.CLOSED
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = State.OPEN

# Usage
circuit_breaker = CircuitBreaker()

try:
    result = circuit_breaker.call(external_api.get_data)
except Exception:
    # Fallback to cache or default value
    result = get_cached_data()
```

### 3. Database Sharding Strategies

**Horizontal Sharding (by user_id):**
```python
def get_shard(user_id, num_shards=10):
    """Consistent hashing"""
    return hash(user_id) % num_shards

# User 123 → Shard 3
# User 456 → Shard 6

# Query routing
user_id = 123
shard = get_shard(user_id)
db = database_connections[shard]
user_data = db.query(f"SELECT * FROM users WHERE user_id = {user_id}")
```

**Range-based Sharding (by date):**
```python
# Orders table sharded by month
def get_orders_table(order_date):
    year_month = order_date.strftime("%Y_%m")
    return f"orders_{year_month}"

# Query for January 2024 orders
table = get_orders_table(datetime(2024, 1, 15))
# table = "orders_2024_01"
```

### 4. Caching Strategies

**Cache-Aside (Lazy Loading):**
```python
def get_user(user_id):
    # Check cache first
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    
    # Cache miss: query database
    user = db.query(f"SELECT * FROM users WHERE id = {user_id}")
    
    # Update cache
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    
    return user
```

**Write-Through:**
```python
def update_user(user_id, data):
    # Write to database
    db.update(f"UPDATE users SET ... WHERE id = {user_id}")
    
    # Update cache immediately
    redis.setex(f"user:{user_id}", 3600, json.dumps(data))
```

**Write-Behind (Async):**
```python
def update_user(user_id, data):
    # Update cache immediately
    redis.setex(f"user:{user_id}", 3600, json.dumps(data))
    
    # Queue database write for later
    queue.publish('db_writes', {
        'table': 'users',
        'id': user_id,
        'data': data
    })
    
    # Background worker processes queue
```

---

## COMMON FOLLOW-UP QUESTIONS

### "How would you handle this at 10x scale?"

**Template Answer:**
```
At 10x scale:

1. Identify bottlenecks:
   - Database: Current 10K QPS → 100K QPS
   - Network: 1 Gbps → 10 Gbps

2. Solutions:
   - Database: Read replicas (3 → 10), sharding, caching
   - Network: CDN for static content, compression
   - Application: Horizontal scaling (10 → 100 instances)

3. Architecture changes:
   - Introduce message queue for async processing
   - Separate read/write paths
   - Implement database sharding

4. Monitoring:
   - Auto-scaling based on CPU/memory
   - Distributed tracing for debugging
   - Cost monitoring

5. Cost estimate:
   - Current: $10K/month
   - At 10x: $60K/month (economies of scale)
```

### "How would you monitor this system?"

**Template Answer:**
```
Four Golden Signals:

1. Latency:
   - p50, p95, p99 response times
   - Alert if p95 >500ms

2. Traffic:
   - Requests per second
   - Active connections

3. Errors:
   - Error rate (4xx, 5xx)
   - Alert if >1% errors

4. Saturation:
   - CPU/memory utilization
   - Database connection pool
   - Alert if >80% utilized

Implementation:
- Prometheus for metrics collection
- Grafana for visualization
- PagerDuty for alerts
- Distributed tracing (Jaeger/X-Ray)

Key dashboards:
- Service health (uptime, error rates)
- Performance (latency percentiles)
- Capacity (resource utilization)
- Business metrics (orders/sec, revenue)
```

---

**THIS COVERS SYSTEM DESIGN FUNDAMENTALS**
**Practice these patterns and you'll be ready for any design question!**