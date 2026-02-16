# ISSUE 3: Connection Pool Leak
## 🔴 Problem Description

### What Happens

**Before Load Test**:
```bash
curl http://localhost:3000/api/cities
# Result: 200 OK ✅
```

**Run Load Test** (10 concurrent connections for 5 seconds):
```bash
npx autocannon -c 10 -d 5 --renderStatusCodes http://localhost:3000/api/cities
# Result: 50 successful, 426 failures
```

**After Load Test**:
```bash
curl http://localhost:3000/api/cities
# Result: 500 Error ❌
# Error: "timeout exceeded when trying to connect"
```

---

### Symptoms

```
Error: timeout exceeded when trying to connect
    at /app/node_modules/pg-pool/index.js:45:11
    at async /app/app.js:59:20
```

**Key Observation**: Single requests worked BEFORE load test but failed AFTER load test. This indicates connections from the load test are NOT being released.

---

## 🔍 Root Cause Analysis

### The Problem

Node.js pg-pool library requires explicit configuration to manage connections properly:

```javascript
// ❌ BROKEN - Default configuration (no pool management)
const pool = new Pool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME
  // Missing: max, idleTimeoutMillis, connectionTimeoutMillis
});
```

**Without proper configuration**:
- ❌ No limit on number of connections
- ❌ Connections stay open indefinitely
- ❌ No timeout for requests waiting for available connection
- ❌ Connection leak accumulates over time

### How Connections Get Exhausted

```
Request 1:  Opens connection → Uses → Should close, but doesn't ❌
Request 2:  Opens connection → Uses → Should close, but doesn't ❌
Request 3:  Opens connection → Uses → Should close, but doesn't ❌
...
Request 50: Opens connection → Uses → Should close, but doesn't ❌

After 50 requests with 10 concurrent connections:
All ~600 PostgreSQL connections are open and stuck
New requests timeout waiting for available connection ❌
```

### Why Database Has 600 Connections

From docker-compose.yml:
```yaml
command: ["postgres", "-c", "max_connections=600"]
```

PostgreSQL allows up to 600 connections. The application opens all 600 during the load test and never closes them, leaving none for new requests.

---

## 📊 Evidence & Symptoms

### Load Test Results
```
Before Fix:
- Latency: 77-196ms (reasonable)
- Throughput: 95 req/sec (stable)
- Responses: 50 success (200), 426 failures (500)
- Success Rate: 10.5%
- Error: "timeout exceeded when trying to connect"

Why Some Succeed & Some Fail:
- First ~50 requests: Find available connections ✅
- Requests 51+: No available connections, timeout ❌
```

### Connection State After Load Test

```
PostgreSQL Connections Status:
- Max allowed: 600
- Current open: ~600 (all exhausted)
- Available to use: ~0
- State: Stuck (never released)

Result: Every new request waits for a connection
        Timeout occurs after 2 seconds
        Request returns 500 error
```