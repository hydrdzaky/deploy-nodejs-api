# Issue 2 : ISSUE #2: API Database Connection Failure

## System Status
```
API Container: ✅ RUNNING
Database Container: ✅ RUNNING
Network: ✅ CONNECTED
API ↔ DB Communication: ❌ FAILED
Service: 🔴 NON-FUNCTIONAL
```

---

## 🔍 Root Cause Analysis

### The Configuration Problem
In `docker-compose.yml`:

```yaml
services:
  api:
    environment:
      - DB_HOST=localhost  # ❌ WRONG - points to API container itself!
      - DB_PORT=5432
      - DB_USER=abc
      - DB_PASSWORD=secret
      - DB_NAME=mydatabase
    depends_on:
      - db
      
  db:
    image: postgres:18.1-alpine3.23
    # Database container runs here on different IP
```

### Why It Failed

#### How Containers Work
```
Host Machine:
  localhost = 127.0.0.1 (the host)

API Container (IP 172.18.0.3):
  localhost = 127.0.0.1 (the API container ITSELF)

Database Container (IP 172.18.0.2):
  localhost = 127.0.0.1 (the database container ITSELF)
```

#### Connection Attempt Flow
```
1. API code: "Connect to localhost:5432"
2. API OS: "localhost = 127.0.0.1 (myself)"
3. Connection: Tries to connect to 127.0.0.1:5432 (within API container)
4. Result: ❌ No PostgreSQL running on API container
5. Error: Connection refused / timeout
```

#### Diagram
```
┌─────────────────────────────────────────────┐
│  Docker Network (app-network)               │
├─────────────────────┬───────────────────────┤
│                     │                       │
│  API Container      │  Database Container   │
│  IP: 172.18.0.3     │  IP: 172.18.0.2      │
│                     │                       │
│  Code: "Connect to  │  PostgreSQL:          │
│  localhost:5432"    │  Running on 5432      │
│                     │                       │
│  Looks at: 127.0.0.1│                       │
│  (itself)           │                       │
│  Not found! ❌      │                       │
└─────────────────────┴───────────────────────┘
```

### Why This Happened
- **Developer mistake**: Confused local machine networking with container networking
- **`localhost` works locally**: On developer's computer, localhost = computer itself
- **Doesn't work in containers**: Inside container, localhost = container itself, NOT other containers
- **No testing in Docker**: Configuration never tested with actual containers
- **Documentation missing**: No explanation of how to connect between containers

---

## ✅ Solution Applied

### The Fix
Change `DB_HOST=localhost` to `DB_HOST=db` (the Docker Compose service name)

### Step-by-Step Fix

#### Step 1: Backup Original File
```bash
cp docker-compose.yml docker-compose.yml.backup.localhost
```

---

#### Step 2: Edit Configuration
```bash
nano docker-compose.yml
```

**Change from**:
```yaml
services:
  api:
    environment:
      - DB_HOST=localhost  # ❌ Wrong
```

**Change to**:
```yaml
services:
  api:
    environment:
      - DB_HOST=db  # ✅ Correct
```

---

#### Step 3: Understand What "db" Means
In `docker-compose.yml`, the service is defined as:
```yaml
services:
  db:  # ← This is the service name
    image: postgres:18.1-alpine3.23
```

**Service name "db" automatically resolves** via Docker DNS to the database container.

---

#### Step 4: Restart Containers
```bash
docker compose down
docker compose up -d
```

**Result**: Containers restart with new configuration

---

#### Step 5: Wait for Database Initialization
```bash
sleep 5  # Give database time to start
```

---

#### Step 6: Verify Configuration
```bash
docker compose logs db | grep "ready to accept"
```

**Expected**:
```
LOG: database system is ready to accept connections
```

---

#### Step 7: Test Connectivity
```bash
curl http://localhost:3000/api/cities
```

**Expected**:
```
{"cities": [{"id":1,"name":"New York"},...]}
HTTP 200 OK ✅
```

---

#### Step 8: Verify Network Connectivity
```bash
docker exec example-nodejs-api-app ping db
```

**Expected**:
```
PING db (172.18.0.2) 56(84) bytes of data
64 bytes from db.example-nodejs-api_default: icmp_seq=1 ttl=64 time=0.123 ms
```

**Status**: ✅ API container can reach database container

---

## 📊 Verification Results

### Before Fix
```
Single Request:    ❌ 500 Error - Connection timeout
Load Test 10 conn: ❌ 926/940 failed (98% error rate)
API ↔ DB:          ❌ Connection refused
```

### After Fix
```
Single Request:    ✅ 200 OK - Data returned
Load Test 10 conn: ⚠️  50/476 successful (10.5% pass)
API ↔ DB:          ✅ Connection successful
```

**Note**: Load test still has 89.5% failures due to Issue #5 (connection pool leak), not this networking issue.

---

## 🌐 How Docker Networking Works

### Docker DNS Service
Docker Compose automatically provides a DNS service:

```
┌──────────────────────────────────────────┐
│  Docker Compose Network                  │
│  (app-network bridge)                    │
├──────────────────────────────────────────┤
│                                          │
│  ┌─────────────────────────────────────┐ │
│  │  Embedded DNS Service               │ │
│  │  - Resolves service names to IPs    │ │
│  │  - Handles load balancing           │ │
│  │  - Works automatically              │ │
│  └─────────────────────────────────────┘ │
│         ▲              ▲                 │
│         │              │                 │
│  ┌──────┴──────┐  ┌────┴──────┐          │
│  │   API       │  │  Database  │         │
│  │ Container   │  │ Container  │         │
│  │             │  │            │         │
│  │ "What's     │  │            │         │
│  │  db?"       │  │            │         │
│  │             │  │            │         │
│  │ DNS: "db =  │  │            │         │
│  │  172.18.0.2"│  │            │         │
│  │             │  │            │         │
│  │ Connect → ──┼─→│ Success! ✅│         │
│  └─────────────┘  └────────────┘         │
│                                          │
└──────────────────────────────────────────┘
```

### Service Name Resolution
```
Step 1: API code requests: "db:5432"
Step 2: Docker DNS intercepts the request
Step 3: Docker DNS: "What's db?"
Step 4: Docker DNS checks docker-compose.yml
Step 5: Found: db service = container example-nodejs-api-db-1
Step 6: Container IP = 172.18.0.2
Step 7: Docker DNS responds: "db = 172.18.0.2"
Step 8: Connection succeeds: 172.18.0.3 → 172.18.0.2:5432
```

---

## 🔄 Why Service Name > Container Name > IP

### Service Name (CORRECT) ✅
```yaml
DB_HOST=db
```

**Advantages**:
- ✅ Stable across container restarts
- ✅ Container name can change, service name stays same
- ✅ Docker DNS handles IP lookup automatically
- ✅ Production standard practice
- ✅ Container IP can change, service name never changes

**Example**:
```
First run:  service "db" → container "example-nodejs-api-db-1"
Second run: service "db" → container "example-nodejs-api-db-2"
           (container name changed, but service name stays "db")
```

---

### Container Name (NOT RECOMMENDED) ⚠️
```yaml
DB_HOST=example-nodejs-api-db-1
```

**Problems**:
- ⚠️ Container name changes when container restarts
- ⚠️ Connection breaks after restart
- ⚠️ Not reliable for production

**Example**:
```
First run:  "example-nodejs-api-db-1" → works
Restart:    Container name becomes "example-nodejs-api-db-2"
            Connection fails! ❌
```

---

### IP Address (NOT RECOMMENDED) ⚠️
```yaml
DB_HOST=172.18.0.2
```

**Problems**:
- ⚠️ Container IP changes when container restarts
- ⚠️ Connection breaks after restart
- ⚠️ Not flexible if network changes

---

### localhost (WRONG) ❌
```yaml
DB_HOST=localhost
```

**Problems**:
- ❌ Refers to API container itself, not database
- ❌ PostgreSQL not running on API container
- ❌ Always fails
- ❌ Common mistake for developers new to containers

---

## 📋 Configuration Details

### Working Configuration
```yaml
services:
  api:
    build: .
    ports:
      - 3000:3000
    environment:
      - NODE_ENV=production
      - PORT=3000
      - DB_HOST=db              # ✅ Service name
      - DB_PORT=5432
      - DB_USER=abc
      - DB_PASSWORD=secret
      - DB_NAME=mydatabase
    depends_on:
      - db

  db:
    image: postgres:18.1-alpine3.23
    ports:
      - 5432:5432
    environment:
      - POSTGRES_USER=abc
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=mydatabase
    volumes:
      - ./init_db.sql:/docker-entrypoint-initdb.d/init_db.sql:ro
```

### Key Settings Explained
| Setting | Value | Why |
|---------|-------|-----|
| `DB_HOST` | `db` | Service name - stable across restarts |
| `DB_PORT` | `5432` | PostgreSQL default port |
| `DB_USER` | `abc` | Database user |
| `DB_PASSWORD` | `secret` | User password (should use .env file) |
| `DB_NAME` | `mydatabase` | Database name |

---

## 🧪 Testing Methods

### Test 1: Single Request
```bash
curl http://localhost:3000/api/cities
```

**Expected**: 200 OK with JSON data  
**If fails**: Check docker-compose.yml DB_HOST value

---

### Test 2: Container Ping
```bash
docker exec example-nodejs-api-api-1 ping db
```

**Expected**: Successful ping to 172.18.0.x  
**If fails**: Service name not resolving (check docker-compose.yml)

---

### Test 3: Port Connectivity
```bash
docker exec example-nodejs-api-api-1 nc -zv db 5432
```

**Expected**: Connection to db 5432 port succeeded  
**If fails**: Port not open or service not running

---

### Test 4: Load Test
```bash
npx autocannon -c 10 -d 5 --renderStatusCodes http://localhost:3000/api/cities
```

**Expected**: Connections successful (Issue #5 may cause timeouts later)  
**If fails**: Either this issue or connection pool issue