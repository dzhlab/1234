# RG System - Troubleshooting Guide

## Версия: 2.13 | Статус: Enterprise Ready | Дата: 2025-11-27

---

## Содержание

1. [Introduction](#1-introduction)
2. [Quick Diagnostic Commands](#2-quick-diagnostic-commands)
3. [Common Issues](#3-common-issues)
4. [System Component Troubleshooting](#4-system-component-troubleshooting)
5. [Performance Issues](#5-performance-issues)
6. [Data Integrity Issues](#6-data-integrity-issues)
7. [Security Incidents](#7-security-incidents)
8. [Deployment Issues](#8-deployment-issues)
9. [Monitoring & Alerting Issues](#9-monitoring--alerting-issues)
10. [Emergency Procedures](#10-emergency-procedures)
11. [Escalation Matrix](#11-escalation-matrix)
12. [Diagnostic Logs & Metrics](#12-diagnostic-logs--metrics)

---

## 1. Introduction

### 1.1. Purpose

Данный документ предоставляет systematic approach к диагностике и решению проблем в RG System.

**Аудитория:**
- Support teams (L1/L2/L3 support)
- DevOps engineers
- SRE teams
- Developers on-call
- System administrators
- Database administrators

### 1.2. How to Use This Guide

**Step 1:** Identify the symptom or error message
**Step 2:** Use Quick Diagnostic Commands для получения базовой информации
**Step 3:** Найдите соответствующую секцию в документе
**Step 4:** Следуйте diagnostic procedures
**Step 5:** Примените решение
**Step 6:** Verify fix
**Step 7:** Document incident (для post-mortem анализа)

### 1.3. Severity Levels

| Severity | Description | Response Time | Examples |
|----------|-------------|---------------|----------|
| **P0 - Critical** | System down, data loss, security breach | Immediate (< 15 min) | Database unreachable, authentication broken, data corruption |
| **P1 - High** | Major functionality impaired | < 1 hour | Scoring algorithm errors, slow API responses (>2s) |
| **P2 - Medium** | Minor functionality impaired | < 4 hours | UI glitches, slow queries, minor bugs |
| **P3 - Low** | Cosmetic issues, feature requests | < 1 business day | Text typos, layout issues |

---

## 2. Quick Diagnostic Commands

### 2.1. System Health Check

```bash
# Overall health status
curl http://localhost:3000/health

# Expected response:
# {"status":"ok","timestamp":"2025-11-27T10:00:00Z","version":"1.0.0"}

# Detailed health check with dependencies
curl http://localhost:3000/health/detailed

# Expected response includes: database, redis, external services status
```

### 2.2. Application Status

```bash
# Check if application is running
ps aux | grep node

# Check application logs (last 100 lines)
tail -n 100 /var/log/rg-system/application.log

# Check for errors in logs (last 1000 lines)
tail -n 1000 /var/log/rg-system/application.log | grep -i error

# Check application metrics
curl http://localhost:3000/metrics | grep -E '(http_requests_total|http_request_duration|db_query_duration)'
```

### 2.3. Database Status

```bash
# Check PostgreSQL status
sudo systemctl status postgresql

# Check database connections
psql -U rg_user -d rg_database -c "SELECT count(*) FROM pg_stat_activity;"

# Check database size
psql -U rg_user -d rg_database -c "SELECT pg_size_pretty(pg_database_size('rg_database'));"

# Check slow queries (queries > 1 second)
psql -U rg_user -d rg_database -c "SELECT pid, now() - pg_stat_activity.query_start AS duration, query FROM pg_stat_activity WHERE (now() - pg_stat_activity.query_start) > interval '1 second' AND state = 'active';"

# Check table sizes
psql -U rg_user -d rg_database -c "SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size FROM pg_tables WHERE schemaname = 'public' ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC LIMIT 10;"
```

### 2.4. Redis Status

```bash
# Check Redis status
sudo systemctl status redis

# Connect to Redis CLI
redis-cli

# Inside Redis CLI:
# Check info
INFO

# Check memory usage
INFO memory

# Check connected clients
CLIENT LIST

# Check keyspace
INFO keyspace

# Monitor commands in real-time
MONITOR
```

### 2.5. Network & Connectivity

```bash
# Check listening ports
sudo netstat -tlnp | grep -E '(3000|5432|6379|9090|3100)'

# Check nginx status
sudo systemctl status nginx

# Test API endpoint
curl -I http://localhost:3000/api/health

# Check DNS resolution
nslookup rg-system.example.com

# Check SSL certificate
openssl s_client -connect rg-system.example.com:443 -servername rg-system.example.com
```

### 2.6. System Resources

```bash
# Check CPU usage
top -b -n 1 | head -20

# Check memory usage
free -h

# Check disk space
df -h

# Check disk I/O
iostat -x 1 5

# Check network traffic
iftop -i eth0
```

---

## 3. Common Issues

### 3.1. "Cannot Connect to Database"

**Symptoms:**
- Error: `ECONNREFUSED 127.0.0.1:5432`
- Error: `Connection terminated unexpectedly`
- Application fails to start

**Diagnostic Steps:**

```bash
# Step 1: Check if PostgreSQL is running
sudo systemctl status postgresql

# Step 2: Check database logs
sudo tail -n 100 /var/log/postgresql/postgresql-14-main.log

# Step 3: Check connection settings
psql -U rg_user -h localhost -d rg_database

# Step 4: Check max connections
psql -U postgres -c "SHOW max_connections;"
psql -U postgres -c "SELECT count(*) FROM pg_stat_activity;"
```

**Common Causes & Solutions:**

| Cause | Solution |
|-------|----------|
| PostgreSQL not running | `sudo systemctl start postgresql` |
| Wrong credentials in .env | Verify `DATABASE_URL` in `.env` file |
| Max connections reached | Increase `max_connections` in `postgresql.conf` or kill idle connections |
| Firewall blocking port 5432 | `sudo ufw allow 5432/tcp` |
| Database disk full | Free up disk space, check with `df -h` |

**Verification:**

```bash
# Application should connect successfully
curl http://localhost:3000/health/detailed | jq '.database.status'
# Expected: "ok"
```

---

### 3.2. "High API Response Time"

**Symptoms:**
- API requests taking > 2 seconds
- Timeout errors
- Users complaining about slow UI

**Diagnostic Steps:**

```bash
# Step 1: Check API metrics
curl http://localhost:3000/metrics | grep http_request_duration_seconds

# Step 2: Check slow queries
psql -U rg_user -d rg_database -c "SELECT pid, now() - pg_stat_activity.query_start AS duration, query FROM pg_stat_activity WHERE (now() - pg_stat_activity.query_start) > interval '1 second' AND state = 'active';"

# Step 3: Check database connection pool
# Look for "pool exhausted" or "waiting for connection" in logs
tail -n 500 /var/log/rg-system/application.log | grep -i pool

# Step 4: Check CPU/Memory
top -b -n 1

# Step 5: Check Redis cache hit rate
redis-cli INFO stats | grep keyspace_hits
redis-cli INFO stats | grep keyspace_misses
```

**Common Causes & Solutions:**

| Cause | Solution |
|-------|----------|
| Missing database indexes | Add indexes on frequently queried columns (see PERFORMANCE_OPTIMIZATION.md) |
| N+1 query problem | Use eager loading with `include` in Prisma queries |
| Large result sets without pagination | Implement pagination (limit/offset) |
| Cache not working | Verify Redis is running, check cache TTL configuration |
| High CPU usage | Scale horizontally (add more application instances) |
| Slow external API calls | Implement timeout, circuit breaker, or cache responses |

**Example Fix - Add Missing Index:**

```sql
-- Check if index exists
SELECT indexname FROM pg_indexes WHERE tablename = 'scores' AND indexname = 'idx_scores_competition_judge';

-- Add index if missing
CREATE INDEX CONCURRENTLY idx_scores_competition_judge ON scores(competition_id, judge_id);

-- Verify index usage
EXPLAIN ANALYZE SELECT * FROM scores WHERE competition_id = 123 AND judge_id = 456;
```

**Verification:**

```bash
# API response time should be < 200ms (p95)
curl -w "@curl-format.txt" -o /dev/null -s http://localhost:3000/api/competitions

# curl-format.txt content:
# time_total: %{time_total}s
```

---

### 3.3. "Authentication Failed"

**Symptoms:**
- Error: `Invalid credentials`
- Error: `JWT token expired`
- Error: `Unauthorized (401)`
- Users cannot log in

**Diagnostic Steps:**

```bash
# Step 1: Check Redis (session storage)
redis-cli PING
# Expected: PONG

# Step 2: Check JWT secret configuration
# Verify JWT_SECRET is set in .env
cat .env | grep JWT_SECRET

# Step 3: Check user exists in database
psql -U rg_user -d rg_database -c "SELECT id, email, role FROM users WHERE email = 'user@example.com';"

# Step 4: Check password hash
psql -U rg_user -d rg_database -c "SELECT id, email, password FROM users WHERE email = 'user@example.com';"

# Step 5: Check authentication logs
tail -n 200 /var/log/rg-system/application.log | grep -i 'auth\|login\|jwt'
```

**Common Causes & Solutions:**

| Cause | Solution |
|-------|----------|
| Wrong password | User should reset password via "Forgot Password" flow |
| JWT token expired | Normal behavior - user needs to refresh token or re-login |
| JWT_SECRET changed | Don't change JWT_SECRET in production (invalidates all tokens) |
| Redis down (sessions lost) | Restart Redis: `sudo systemctl restart redis` |
| Account locked (failed attempts) | Unlock account: `UPDATE users SET login_attempts = 0, locked_until = NULL WHERE email = 'user@example.com';` |
| Clock skew (JWT validation) | Sync server time: `sudo ntpdate -s time.nist.gov` |

**Example Fix - Unlock Account:**

```sql
-- Check if account is locked
SELECT email, login_attempts, locked_until FROM users WHERE email = 'user@example.com';

-- Unlock account
UPDATE users SET login_attempts = 0, locked_until = NULL WHERE email = 'user@example.com';
```

**Verification:**

```bash
# Test login
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"password123"}'

# Expected: {"token":"eyJhbG...", "user":{...}}
```

---

### 3.4. "Scoring Calculation Incorrect"

**Symptoms:**
- Final score doesn't match expected value
- Deductions not applied correctly
- Bonus points missing

**Diagnostic Steps:**

```bash
# Step 1: Get detailed score breakdown
curl http://localhost:3000/api/scores/12345/breakdown

# Step 2: Check scoring algorithm version
psql -U rg_user -d rg_database -c "SELECT version FROM scoring_algorithms WHERE active = true;"

# Step 3: Check score components in database
psql -U rg_user -d rg_database -c "SELECT * FROM scores WHERE id = 12345;"
psql -U rg_user -d rg_database -c "SELECT * FROM score_components WHERE score_id = 12345;"

# Step 4: Recalculate score manually using algorithm
# See SCORING_ALGORITHMS.md for calculation formulas

# Step 5: Check audit logs for score modifications
psql -U rg_user -d rg_database -c "SELECT * FROM audit_log WHERE entity_type = 'score' AND entity_id = '12345' ORDER BY created_at DESC;"
```

**Common Causes & Solutions:**

| Cause | Solution |
|-------|----------|
| Wrong difficulty coefficient | Verify difficulty (D1-D6) is correct for the exercise |
| Execution score input error | Judge should review and re-submit if needed |
| Deduction calculation bug | Check deduction logic in `calculateDeductions()` function |
| Bonus not configured | Configure bonus rules in competition settings |
| Rounding error | Verify rounding is done per FIG rules (0.05 precision) |
| Cached old value | Clear cache: `redis-cli DEL score:12345` |

**Example Fix - Recalculate Score:**

```typescript
// Trigger score recalculation via API
curl -X POST http://localhost:3000/api/scores/12345/recalculate \
  -H "Authorization: Bearer <admin-token>"

// Or via database trigger
UPDATE scores SET updated_at = NOW() WHERE id = 12345;
```

**Verification:**

```bash
# Check score breakdown matches expected calculation
curl http://localhost:3000/api/scores/12345/breakdown | jq

# Verify with second judge or chief judge
```

---

### 3.5. "File Upload Failed"

**Symptoms:**
- Error: `File too large`
- Error: `Invalid file type`
- Upload hangs or times out

**Diagnostic Steps:**

```bash
# Step 1: Check file size limit
cat .env | grep MAX_FILE_SIZE

# Step 2: Check upload directory permissions
ls -la /var/uploads/rg-system

# Step 3: Check disk space
df -h /var/uploads

# Step 4: Check nginx upload size limit
grep client_max_body_size /etc/nginx/nginx.conf

# Step 5: Check application logs
tail -n 100 /var/log/rg-system/application.log | grep -i upload
```

**Common Causes & Solutions:**

| Cause | Solution |
|-------|----------|
| File exceeds size limit | Increase `MAX_FILE_SIZE` in `.env` (default 10MB) |
| Nginx blocking large uploads | Increase `client_max_body_size` in nginx.conf |
| Upload directory full | Free up disk space or configure different upload path |
| Wrong file type | Check allowed MIME types in `ALLOWED_FILE_TYPES` |
| Insufficient permissions | `sudo chown -R app:app /var/uploads/rg-system` |
| Timeout on slow network | Increase nginx `client_body_timeout` |

**Example Fix - Increase Upload Limit:**

```bash
# Update .env
echo "MAX_FILE_SIZE=52428800" >> .env  # 50MB

# Update nginx.conf
sudo nano /etc/nginx/nginx.conf
# Add: client_max_body_size 50M;

# Restart services
sudo systemctl restart nginx
pm2 restart rg-system
```

**Verification:**

```bash
# Test file upload
curl -X POST http://localhost:3000/api/upload \
  -H "Authorization: Bearer <token>" \
  -F "file=@test-video.mp4"

# Expected: {"url":"https://cdn.example.com/uploads/xxx.mp4"}
```

---

### 3.6. "Frontend Not Loading"

**Symptoms:**
- Blank white screen
- Error: `Failed to load resource`
- Console errors in browser

**Diagnostic Steps:**

```bash
# Step 1: Check if frontend build exists
ls -la /var/www/rg-system/dist/

# Step 2: Check nginx configuration
sudo nginx -t

# Step 3: Check nginx logs
sudo tail -n 100 /var/log/nginx/error.log

# Step 4: Check browser console (F12)
# Look for JavaScript errors, network errors

# Step 5: Check API connectivity from frontend
curl http://localhost:3000/api/health
```

**Common Causes & Solutions:**

| Cause | Solution |
|-------|----------|
| Build not deployed | Run `npm run build` and copy to `/var/www/rg-system/dist/` |
| Nginx not serving static files | Check `root` directive in nginx config |
| CORS errors | Configure CORS headers in backend (see API_DOCUMENTATION.md) |
| JavaScript error | Check browser console, review recent code changes |
| API URL misconfigured | Verify `VITE_API_URL` in frontend `.env` |
| Cache issue | Hard refresh browser (Ctrl+Shift+R) or clear cache |

**Example Fix - CORS Configuration:**

```typescript
// backend/src/middleware/cors.ts
app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:5173',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

**Verification:**

```bash
# Frontend should load successfully
curl -I http://localhost/

# Check for CORS headers
curl -I http://localhost:3000/api/health \
  -H "Origin: http://localhost:5173"

# Expected: Access-Control-Allow-Origin header present
```

---

## 4. System Component Troubleshooting

### 4.1. Database (PostgreSQL)

#### 4.1.1. Database Connection Pool Exhausted

**Symptoms:**
```
Error: Timed out fetching a new connection from the connection pool
Error: Connection pool exhausted
```

**Diagnosis:**

```bash
# Check current connections
psql -U rg_user -d rg_database -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"

# Check pool configuration
cat .env | grep DATABASE_POOL

# Check long-running transactions
psql -U rg_user -d rg_database -c "SELECT pid, now() - xact_start AS duration, state, query FROM pg_stat_activity WHERE xact_start IS NOT NULL ORDER BY duration DESC LIMIT 10;"
```

**Solution:**

```bash
# Option 1: Increase pool size (if CPU/memory allows)
# Edit .env
DATABASE_POOL_MIN=10
DATABASE_POOL_MAX=50

# Option 2: Kill idle connections
psql -U postgres -d rg_database -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle' AND state_change < now() - interval '1 hour';"

# Option 3: Set connection timeout
# Edit postgresql.conf
idle_in_transaction_session_timeout = 600000  # 10 minutes
```

#### 4.1.2. Slow Queries

**Diagnosis:**

```bash
# Enable query logging (postgresql.conf)
log_min_duration_statement = 1000  # Log queries > 1 second

# Check slow query log
sudo tail -n 500 /var/log/postgresql/postgresql-14-main.log | grep "duration:"

# Analyze specific query
psql -U rg_user -d rg_database

EXPLAIN ANALYZE
SELECT s.*, g.name as gymnast_name
FROM scores s
JOIN gymnasts g ON s.gymnast_id = g.id
WHERE s.competition_id = 123
ORDER BY s.final_score DESC;
```

**Solution:**

```sql
-- Add missing indexes
CREATE INDEX CONCURRENTLY idx_scores_competition_id ON scores(competition_id);
CREATE INDEX CONCURRENTLY idx_scores_final_score ON scores(final_score DESC);

-- Update statistics
ANALYZE scores;

-- Consider partitioning for large tables
-- See DATABASE_SCHEMA.md for partitioning strategy
```

#### 4.1.3. Database Locks

**Diagnosis:**

```sql
-- Check for locks
SELECT
  blocked_locks.pid AS blocked_pid,
  blocked_activity.usename AS blocked_user,
  blocking_locks.pid AS blocking_pid,
  blocking_activity.usename AS blocking_user,
  blocked_activity.query AS blocked_statement,
  blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
  AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
  AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
  AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
  AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
  AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
  AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
  AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
  AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
  AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

**Solution:**

```sql
-- Terminate blocking query (use carefully!)
SELECT pg_terminate_backend(12345);  -- Replace with blocking_pid

-- Prevent locks by using shorter transactions
-- Avoid long-running transactions during peak hours
```

---

### 4.2. Redis Cache

#### 4.2.1. Redis Out of Memory

**Symptoms:**
```
Error: OOM command not allowed when used memory > 'maxmemory'
```

**Diagnosis:**

```bash
redis-cli INFO memory

# Check memory usage
redis-cli INFO memory | grep used_memory_human

# Check eviction stats
redis-cli INFO stats | grep evicted_keys

# Check maxmemory setting
redis-cli CONFIG GET maxmemory
```

**Solution:**

```bash
# Option 1: Increase maxmemory (if server has capacity)
redis-cli CONFIG SET maxmemory 2gb
# Make permanent in redis.conf

# Option 2: Enable eviction policy
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# Option 3: Clear old/unused keys
redis-cli --scan --pattern "session:*" | xargs redis-cli DEL

# Option 4: Flush specific cache namespace
redis-cli --scan --pattern "cache:old:*" | xargs redis-cli DEL
```

#### 4.2.2. Cache Miss Rate Too High

**Diagnosis:**

```bash
# Check hit/miss ratio
redis-cli INFO stats | grep keyspace

# Calculate hit rate:
# hit_rate = keyspace_hits / (keyspace_hits + keyspace_misses)

# Check TTL of keys
redis-cli TTL "score:12345"
redis-cli TTL "leaderboard:competition:123"

# Monitor cache operations
redis-cli MONITOR
```

**Solution:**

```bash
# Increase TTL for frequently accessed data
# Edit cache configuration
CACHE_TTL_SCORES=3600        # 1 hour
CACHE_TTL_LEADERBOARD=300    # 5 minutes
CACHE_TTL_USER_SESSION=1800  # 30 minutes

# Implement cache warming for critical data
curl -X POST http://localhost:3000/api/admin/cache/warm

# Pre-populate cache before peak hours
```

---

### 4.3. Application Server (Node.js)

#### 4.3.1. Memory Leak

**Symptoms:**
- Memory usage continuously increasing
- Application crashes with `FATAL ERROR: JavaScript heap out of memory`

**Diagnosis:**

```bash
# Check Node.js memory usage
pm2 monit

# Take heap snapshot
node --inspect index.js
# Then use Chrome DevTools → Memory → Take Heap Snapshot

# Check for memory leaks with clinic
npm install -g clinic
clinic doctor -- node index.js

# Monitor memory over time
watch -n 5 "pm2 info rg-system | grep memory"
```

**Solution:**

```bash
# Option 1: Increase heap size
NODE_OPTIONS="--max-old-space-size=4096" pm2 start index.js

# Option 2: Restart application periodically (workaround)
pm2 start index.js --cron-restart="0 4 * * *"  # Restart daily at 4 AM

# Option 3: Fix memory leak in code
# Common causes:
# - Event listeners not removed
# - Large objects in global scope
# - Circular references
# - Unclosed database connections
```

#### 4.3.2. High CPU Usage

**Diagnosis:**

```bash
# Check CPU usage
top -p $(pgrep -f "node.*index.js")

# Profile CPU usage
node --prof index.js
# Then analyze with: node --prof-process isolate-*.log

# Use clinic flame
clinic flame -- node index.js
```

**Solution:**

```javascript
// Optimize hot paths in code

// Example: Use Set instead of Array for lookups
// Before (O(n)):
const judges = [1, 2, 3, 4, 5];
if (judges.includes(judgeId)) { ... }

// After (O(1)):
const judgesSet = new Set([1, 2, 3, 4, 5]);
if (judgesSet.has(judgeId)) { ... }

// Example: Cache expensive calculations
const memoize = require('lodash/memoize');
const calculateScore = memoize((difficulty, execution, deductions) => {
  // expensive calculation
  return (difficulty + execution - deductions);
});
```

---

### 4.4. Frontend (React)

#### 4.4.1. Slow Rendering

**Diagnosis:**

```javascript
// Use React DevTools Profiler
// 1. Open React DevTools
// 2. Go to Profiler tab
// 3. Click record, perform action, stop recording
// 4. Analyze component render times

// Add performance marks in code
performance.mark('start-render');
// ... render logic
performance.mark('end-render');
performance.measure('render-time', 'start-render', 'end-render');
console.log(performance.getEntriesByName('render-time')[0].duration);
```

**Solution:**

```typescript
// Use React.memo for expensive components
const LeaderboardRow = React.memo(({ score }) => {
  return <tr>...</tr>;
});

// Use useMemo for expensive calculations
const sortedScores = useMemo(() => {
  return scores.sort((a, b) => b.finalScore - a.finalScore);
}, [scores]);

// Use useCallback for event handlers
const handleScoreChange = useCallback((id, value) => {
  updateScore(id, value);
}, [updateScore]);

// Virtualize long lists
import { FixedSizeList } from 'react-window';

<FixedSizeList
  height={600}
  itemCount={scores.length}
  itemSize={50}
>
  {({ index, style }) => (
    <div style={style}>
      <LeaderboardRow score={scores[index]} />
    </div>
  )}
</FixedSizeList>
```

#### 4.4.2. State Management Issues

**Common Issues:**

```typescript
// Problem: Stale state in useEffect
useEffect(() => {
  fetchData(userId);  // userId might be stale
}, []);  // Empty dependency array

// Solution: Include dependencies
useEffect(() => {
  fetchData(userId);
}, [userId, fetchData]);


// Problem: Race condition in async state updates
const [data, setData] = useState(null);
useEffect(() => {
  fetchData().then(setData);  // Can cause race condition
}, [query]);

// Solution: Use cleanup function
useEffect(() => {
  let cancelled = false;
  fetchData(query).then(result => {
    if (!cancelled) setData(result);
  });
  return () => { cancelled = true; };
}, [query]);


// Problem: Unnecessary re-renders
const [count, setCount] = useState(0);
setCount(count + 1);  // Triggers re-render even if count doesn't change

// Solution: Use functional update
setCount(prev => prev + 1);
```

---

## 5. Performance Issues

### 5.1. Database Query Performance

**Diagnostic Query:**

```sql
-- Top 10 slowest queries
SELECT
  query,
  calls,
  total_time,
  mean_time,
  max_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- Enable pg_stat_statements if not already enabled
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

**Optimization Techniques:**

```sql
-- 1. Add indexes on foreign keys
CREATE INDEX CONCURRENTLY idx_scores_gymnast_id ON scores(gymnast_id);
CREATE INDEX CONCURRENTLY idx_scores_judge_id ON scores(judge_id);
CREATE INDEX CONCURRENTLY idx_scores_competition_id ON scores(competition_id);

-- 2. Add composite indexes for common query patterns
CREATE INDEX CONCURRENTLY idx_scores_competition_gymnast
ON scores(competition_id, gymnast_id)
INCLUDE (final_score, difficulty, execution);

-- 3. Add partial indexes for filtered queries
CREATE INDEX CONCURRENTLY idx_scores_pending
ON scores(status)
WHERE status = 'pending';

-- 4. Use covering indexes to avoid table lookups
CREATE INDEX CONCURRENTLY idx_gymnasts_name_dob
ON gymnasts(last_name, first_name)
INCLUDE (date_of_birth, country);
```

### 5.2. API Endpoint Performance

**Monitoring:**

```bash
# Check endpoint response times in Prometheus
curl http://localhost:9090/api/v1/query?query='histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))'

# Check in application metrics
curl http://localhost:3000/metrics | grep http_request_duration_seconds
```

**Optimization:**

```typescript
// 1. Implement response caching
import { cacheMiddleware } from './middleware/cache';

app.get('/api/leaderboard/:competitionId',
  cacheMiddleware({ ttl: 60 }), // Cache for 60 seconds
  async (req, res) => {
    // ... handler logic
  }
);

// 2. Add pagination
app.get('/api/scores', async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const offset = (page - 1) * limit;

  const scores = await prisma.score.findMany({
    skip: offset,
    take: limit,
    orderBy: { createdAt: 'desc' }
  });

  res.json({ scores, page, limit });
});

// 3. Use database projections (select only needed fields)
const scores = await prisma.score.findMany({
  select: {
    id: true,
    finalScore: true,
    gymnast: {
      select: { firstName: true, lastName: true }
    }
  }
});

// 4. Implement N+1 query prevention with eager loading
const competitions = await prisma.competition.findMany({
  include: {
    scores: {
      include: {
        gymnast: true,
        judge: true
      }
    }
  }
});
```

### 5.3. Frontend Performance

**Bundle Size Optimization:**

```bash
# Analyze bundle size
npm run build -- --analyze

# Use code splitting
# Before: import entire library
import _ from 'lodash';

# After: import only needed functions
import debounce from 'lodash/debounce';

# Lazy load routes
const AdminPanel = lazy(() => import('./pages/AdminPanel'));

<Suspense fallback={<Loading />}>
  <Route path="/admin" element={<AdminPanel />} />
</Suspense>
```

**Image Optimization:**

```typescript
// Use responsive images
<img
  src="/images/gymnast-small.jpg"
  srcSet="/images/gymnast-small.jpg 400w,
          /images/gymnast-medium.jpg 800w,
          /images/gymnast-large.jpg 1200w"
  sizes="(max-width: 600px) 400px,
         (max-width: 1200px) 800px,
         1200px"
  alt="Gymnast"
  loading="lazy"
/>

// Use WebP format with fallback
<picture>
  <source srcSet="/images/gymnast.webp" type="image/webp" />
  <source srcSet="/images/gymnast.jpg" type="image/jpeg" />
  <img src="/images/gymnast.jpg" alt="Gymnast" />
</picture>
```

---

## 6. Data Integrity Issues

### 6.1. Duplicate Records

**Detection:**

```sql
-- Find duplicate gymnasts (same name and DOB)
SELECT first_name, last_name, date_of_birth, COUNT(*)
FROM gymnasts
GROUP BY first_name, last_name, date_of_birth
HAVING COUNT(*) > 1;

-- Find duplicate scores (same gymnast, competition, apparatus)
SELECT gymnast_id, competition_id, apparatus, COUNT(*)
FROM scores
GROUP BY gymnast_id, competition_id, apparatus
HAVING COUNT(*) > 1;
```

**Prevention:**

```sql
-- Add unique constraint
ALTER TABLE scores
ADD CONSTRAINT unique_score_per_gymnast_apparatus
UNIQUE (gymnast_id, competition_id, apparatus, judge_id);

-- Add unique index
CREATE UNIQUE INDEX idx_gymnasts_unique
ON gymnasts(first_name, last_name, date_of_birth);
```

**Cleanup:**

```sql
-- Keep only the latest duplicate score
WITH duplicates AS (
  SELECT id,
    ROW_NUMBER() OVER (
      PARTITION BY gymnast_id, competition_id, apparatus
      ORDER BY created_at DESC
    ) as rn
  FROM scores
)
DELETE FROM scores
WHERE id IN (
  SELECT id FROM duplicates WHERE rn > 1
);
```

### 6.2. Orphaned Records

**Detection:**

```sql
-- Find scores without gymnast
SELECT * FROM scores s
LEFT JOIN gymnasts g ON s.gymnast_id = g.id
WHERE g.id IS NULL;

-- Find scores without competition
SELECT * FROM scores s
LEFT JOIN competitions c ON s.competition_id = c.id
WHERE c.id IS NULL;
```

**Cleanup:**

```sql
-- Delete orphaned scores
DELETE FROM scores
WHERE gymnast_id NOT IN (SELECT id FROM gymnasts);

DELETE FROM scores
WHERE competition_id NOT IN (SELECT id FROM competitions);
```

**Prevention:**

```sql
-- Add foreign key constraints (should already exist)
ALTER TABLE scores
ADD CONSTRAINT fk_scores_gymnast
FOREIGN KEY (gymnast_id) REFERENCES gymnasts(id)
ON DELETE CASCADE;

ALTER TABLE scores
ADD CONSTRAINT fk_scores_competition
FOREIGN KEY (competition_id) REFERENCES competitions(id)
ON DELETE CASCADE;
```

### 6.3. Data Corruption

**Detection:**

```sql
-- Check for invalid score values
SELECT * FROM scores
WHERE final_score < 0 OR final_score > 20;

SELECT * FROM scores
WHERE difficulty < 0 OR difficulty > 10;

SELECT * FROM scores
WHERE execution < 0 OR execution > 10;

-- Check for invalid dates
SELECT * FROM competitions
WHERE end_date < start_date;

SELECT * FROM gymnasts
WHERE date_of_birth > CURRENT_DATE;
```

**Fix:**

```sql
-- Add check constraints
ALTER TABLE scores
ADD CONSTRAINT check_final_score_range
CHECK (final_score >= 0 AND final_score <= 20);

ALTER TABLE scores
ADD CONSTRAINT check_difficulty_range
CHECK (difficulty >= 0 AND difficulty <= 10);

ALTER TABLE competitions
ADD CONSTRAINT check_dates
CHECK (end_date >= start_date);
```

---

## 7. Security Incidents

### 7.1. Suspicious Login Activity

**Detection:**

```sql
-- Multiple failed login attempts
SELECT email, COUNT(*) as failed_attempts, MAX(created_at) as last_attempt
FROM audit_log
WHERE action = 'login_failed'
  AND created_at > NOW() - INTERVAL '1 hour'
GROUP BY email
HAVING COUNT(*) >= 5
ORDER BY failed_attempts DESC;

-- Logins from unusual locations
SELECT user_id, ip_address, location, created_at
FROM audit_log
WHERE action = 'login_success'
  AND created_at > NOW() - INTERVAL '24 hours'
  AND location NOT IN (
    SELECT DISTINCT location
    FROM audit_log
    WHERE user_id = audit_log.user_id
      AND created_at < NOW() - INTERVAL '7 days'
  );
```

**Response:**

```sql
-- Lock suspicious accounts
UPDATE users
SET locked_until = NOW() + INTERVAL '24 hours',
    lock_reason = 'Suspicious activity detected'
WHERE email IN (
  SELECT email FROM audit_log
  WHERE action = 'login_failed'
    AND created_at > NOW() - INTERVAL '1 hour'
  GROUP BY email
  HAVING COUNT(*) >= 10
);

-- Invalidate all sessions for user
DELETE FROM sessions WHERE user_id = 12345;

-- Force password reset
UPDATE users
SET force_password_reset = true
WHERE id = 12345;
```

### 7.2. SQL Injection Attempt

**Detection:**

```bash
# Check application logs for SQL injection patterns
grep -E "(UNION SELECT|DROP TABLE|'; DROP|1=1|admin'--)" /var/log/rg-system/application.log

# Check for suspicious characters in input
grep -E "(%27|%20|%2D%2D|%23)" /var/log/nginx/access.log
```

**Prevention:**

```typescript
// ALWAYS use parameterized queries (Prisma automatically does this)
// ✅ SAFE
const user = await prisma.user.findUnique({
  where: { email: userInput }
});

// ❌ NEVER use raw SQL with user input
const result = await prisma.$queryRaw`
  SELECT * FROM users WHERE email = ${userInput}
`; // This is vulnerable!

// ✅ If you must use raw SQL, use parameterized queries
const result = await prisma.$queryRaw`
  SELECT * FROM users WHERE email = ${userInput}
`; // Prisma still protects you here
```

### 7.3. XSS Attack

**Detection:**

```bash
# Check for script tags in database
psql -U rg_user -d rg_database -c "SELECT * FROM gymnasts WHERE first_name LIKE '%<script%' OR last_name LIKE '%<script%';"

psql -U rg_user -d rg_database -c "SELECT * FROM competitions WHERE name LIKE '%<script%' OR description LIKE '%<script%';"
```

**Cleanup:**

```sql
-- Remove malicious scripts (backup first!)
UPDATE gymnasts
SET first_name = regexp_replace(first_name, '<script[^>]*>.*?</script>', '', 'gi')
WHERE first_name ~* '<script';
```

**Prevention:**

```typescript
// ✅ React automatically escapes output (use JSX)
<div>{userInput}</div>  // Safe

// ❌ NEVER use dangerouslySetInnerHTML with user input
<div dangerouslySetInnerHTML={{ __html: userInput }} />  // Vulnerable!

// ✅ If you must use HTML, sanitize it
import DOMPurify from 'dompurify';

<div dangerouslySetInnerHTML={{
  __html: DOMPurify.sanitize(userInput)
}} />

// Backend validation
import validator from 'validator';

const sanitizedName = validator.escape(req.body.name);
```

### 7.4. Data Breach

**Immediate Actions:**

```bash
# 1. Isolate affected system
sudo ufw deny from any to any

# 2. Preserve evidence
sudo tar -czf /backup/forensics-$(date +%Y%m%d-%H%M%S).tar.gz \
  /var/log/rg-system/ \
  /var/log/postgresql/ \
  /var/log/nginx/

# 3. Review access logs
tail -n 10000 /var/log/nginx/access.log > /backup/access-log-snapshot.txt

# 4. Check for unauthorized database access
psql -U postgres -d rg_database -c "SELECT * FROM pg_stat_activity;"

# 5. Rotate all credentials
# - Database passwords
# - JWT secrets
# - API keys
# - SSL certificates
```

**Notification Procedure:**

1. **Immediate** (< 1 hour): Notify security team and management
2. **24 hours**: Assess scope of breach (what data was accessed)
3. **72 hours**: Notify affected users (if PII was compromised)
4. **GDPR**: Report to data protection authority within 72 hours (if applicable)

---

## 8. Deployment Issues

### 8.1. Database Migration Failed

**Symptoms:**
```
Error: Migration failed
Error: Relation already exists
Error: Column does not exist
```

**Diagnosis:**

```bash
# Check migration status
npx prisma migrate status

# Check which migrations have been applied
psql -U rg_user -d rg_database -c "SELECT * FROM _prisma_migrations ORDER BY finished_at DESC LIMIT 10;"

# Check for failed migrations
psql -U rg_user -d rg_database -c "SELECT * FROM _prisma_migrations WHERE finished_at IS NULL OR logs IS NOT NULL;"
```

**Solution:**

```bash
# Option 1: Resolve and retry migration
npx prisma migrate resolve --applied <migration-name>
npx prisma migrate deploy

# Option 2: Mark migration as rolled back
npx prisma migrate resolve --rolled-back <migration-name>

# Option 3: Manual fix (advanced)
psql -U rg_user -d rg_database

-- Fix the issue manually, then mark migration as applied
INSERT INTO _prisma_migrations (id, checksum, finished_at, migration_name, logs, rolled_back_at, started_at, applied_steps_count)
VALUES ('xxx', 'xxx', NOW(), '20231127_add_column', NULL, NULL, NOW(), 1);
```

### 8.2. Zero-Downtime Deployment Failed

**Symptoms:**
- Requests failing during deployment
- Database connection errors
- Version mismatch errors

**Diagnosis:**

```bash
# Check running instances
pm2 list

# Check instance versions
curl http://instance1:3000/health | jq '.version'
curl http://instance2:3000/health | jq '.version'

# Check load balancer status
sudo nginx -t
sudo systemctl status nginx
```

**Solution:**

```bash
# Blue-Green deployment strategy

# Step 1: Start new instances (green)
pm2 start ecosystem.config.js --name rg-system-green

# Step 2: Health check new instances
curl http://localhost:3001/health

# Step 3: Update nginx to point to green instances
sudo nano /etc/nginx/conf.d/rg-system.conf
# Update upstream to point to green instances

sudo nginx -t && sudo systemctl reload nginx

# Step 4: Monitor for errors
tail -f /var/log/nginx/error.log

# Step 5: Stop old instances (blue) after verification
pm2 stop rg-system-blue
pm2 delete rg-system-blue
```

### 8.3. Rollback Procedure

**When to Rollback:**
- Critical bug in production
- Data corruption detected
- Performance degradation > 50%
- Security vulnerability introduced

**Rollback Steps:**

```bash
# 1. Revert application code
git revert <commit-hash>
git push origin main

# 2. Rebuild and deploy
npm run build
pm2 restart rg-system

# 3. Rollback database migration (if applicable)
npx prisma migrate resolve --rolled-back <migration-name>

# Or restore database from backup
psql -U rg_user -d rg_database < /backup/rg_database_backup_before_deploy.sql

# 4. Clear cache to remove new version artifacts
redis-cli FLUSHALL

# 5. Verify rollback
curl http://localhost:3000/health | jq '.version'
# Should show previous version

# 6. Monitor logs for errors
tail -f /var/log/rg-system/application.log
```

---

## 9. Monitoring & Alerting Issues

### 9.1. Prometheus Not Scraping Metrics

**Diagnosis:**

```bash
# Check Prometheus targets
curl http://localhost:9090/api/v1/targets | jq

# Check application metrics endpoint
curl http://localhost:3000/metrics

# Check Prometheus configuration
cat /etc/prometheus/prometheus.yml

# Check Prometheus logs
sudo tail -f /var/log/prometheus/prometheus.log
```

**Solution:**

```yaml
# Fix prometheus.yml configuration
scrape_configs:
  - job_name: 'rg-system'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:3000']
    metrics_path: '/metrics'
    scheme: 'http'

# Reload Prometheus configuration
curl -X POST http://localhost:9090/-/reload

# Or restart Prometheus
sudo systemctl restart prometheus
```

### 9.2. Alerts Not Firing

**Diagnosis:**

```bash
# Check AlertManager status
curl http://localhost:9093/api/v1/status | jq

# Check alert rules in Prometheus
curl http://localhost:9090/api/v1/rules | jq

# Check if alerts are pending/firing
curl http://localhost:9090/api/v1/alerts | jq

# Test alert
curl -X POST http://localhost:9093/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d '[{
    "labels": {"alertname": "TestAlert", "severity": "critical"},
    "annotations": {"summary": "Test alert"},
    "startsAt": "2025-11-27T10:00:00Z"
  }]'
```

**Solution:**

```yaml
# Check alert rule syntax (prometheus.rules.yml)
groups:
  - name: rg_system_alerts
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} errors/sec"

# Validate rules
promtool check rules prometheus.rules.yml

# Reload Prometheus
curl -X POST http://localhost:9090/-/reload
```

### 9.3. Grafana Dashboard Not Showing Data

**Diagnosis:**

```bash
# Check Grafana logs
sudo tail -f /var/log/grafana/grafana.log

# Test Prometheus datasource
curl -X GET http://localhost:3000/api/datasources

# Test query
curl -X POST http://localhost:3000/api/ds/query \
  -H "Content-Type: application/json" \
  -d '{
    "queries": [{
      "refId": "A",
      "expr": "up{job=\"rg-system\"}"
    }]
  }'
```

**Solution:**

```bash
# Verify Prometheus datasource configuration in Grafana
# Settings → Data Sources → Prometheus
# URL: http://localhost:9090
# Access: Server (default)

# Import dashboard from file
curl -X POST http://admin:admin@localhost:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d @dashboard.json

# Or reset Grafana database
sudo systemctl stop grafana-server
sudo rm /var/lib/grafana/grafana.db
sudo systemctl start grafana-server
```

---

## 10. Emergency Procedures

### 10.1. System Down (P0)

**Immediate Actions (First 5 minutes):**

```bash
# 1. Acknowledge incident
echo "Incident acknowledged at $(date)" >> /var/log/incidents.log

# 2. Check if application is running
pm2 list

# 3. Check critical dependencies
sudo systemctl status postgresql
sudo systemctl status redis
sudo systemctl status nginx

# 4. Check health endpoint
curl http://localhost:3000/health

# 5. Notify on-call team
# Use PagerDuty, Slack, or phone
```

**Recovery Steps:**

```bash
# Step 1: Restart application
pm2 restart rg-system

# Step 2: If database is down
sudo systemctl start postgresql
# Wait 30 seconds for database to be ready
sleep 30
pm2 restart rg-system

# Step 3: If Redis is down
sudo systemctl start redis
pm2 restart rg-system

# Step 4: Check logs for errors
tail -n 500 /var/log/rg-system/application.log | grep -i error

# Step 5: Verify system is operational
curl http://localhost:3000/health
curl http://localhost:3000/api/competitions

# Step 6: Monitor for 15 minutes
watch -n 10 "curl -s http://localhost:3000/health | jq"
```

### 10.2. Data Loss (P0)

**Immediate Actions:**

```bash
# 1. STOP all writes immediately
# Put application in read-only mode or shut down
pm2 stop rg-system

# 2. Assess scope of data loss
psql -U rg_user -d rg_database -c "SELECT COUNT(*) FROM scores WHERE created_at > '2025-11-27';"

# 3. Check if data exists in backups
ls -lh /backup/postgresql/

# 4. Notify stakeholders
echo "Data loss incident - $(date)" | mail -s "P0 INCIDENT" team@example.com
```

**Recovery Steps:**

```bash
# Option 1: Restore from latest backup
sudo systemctl stop postgresql

# Restore database
sudo -u postgres psql -c "DROP DATABASE rg_database;"
sudo -u postgres psql -c "CREATE DATABASE rg_database;"
sudo -u postgres psql rg_database < /backup/postgresql/rg_database_2025-11-27_03-00.sql

sudo systemctl start postgresql

# Option 2: Point-in-Time Recovery (PITR)
# See OPERATIONS_MANUAL.md section 6.3 for detailed PITR procedure

# Verify data restoration
psql -U rg_user -d rg_database -c "SELECT COUNT(*) FROM scores;"

# Restart application
pm2 start rg-system

# Verify system functionality
curl http://localhost:3000/health
```

### 10.3. Security Breach (P0)

**Immediate Actions:**

```bash
# 1. Isolate compromised system
sudo ufw deny from any to any
# Or take server offline

# 2. Preserve evidence
sudo cp -r /var/log /backup/incident-$(date +%Y%m%d-%H%M%S)/

# 3. Revoke all access tokens
redis-cli FLUSHDB  # Clear all sessions

# 4. Change all credentials
# - Database passwords
# - JWT secrets
# - API keys

# 5. Notify security team and legal
```

**Investigation:**

```bash
# Check access logs for suspicious activity
grep -E "POST|DELETE|PUT" /var/log/nginx/access.log | grep -v "200\|201\|204"

# Check database audit log
psql -U rg_user -d rg_database -c "SELECT * FROM audit_log ORDER BY created_at DESC LIMIT 1000;"

# Check for unauthorized database changes
psql -U rg_user -d rg_database -c "SELECT * FROM pg_stat_user_tables WHERE n_tup_ins > 0 OR n_tup_upd > 0 OR n_tup_del > 0 ORDER BY last_mod_time DESC;"

# Check for privilege escalation
psql -U rg_user -d rg_database -c "SELECT * FROM users WHERE role = 'admin' ORDER BY updated_at DESC;"
```

### 10.4. Database Corruption (P0)

**Detection:**

```bash
# Check for database corruption
sudo -u postgres pg_checksums --check -D /var/lib/postgresql/14/main

# Check table integrity
psql -U postgres -d rg_database -c "
  SELECT tablename FROM pg_tables WHERE schemaname = 'public';
" | xargs -I {} psql -U postgres -d rg_database -c "VACUUM ANALYZE {};"
```

**Recovery:**

```bash
# Option 1: Repair corruption (if minor)
psql -U postgres -d rg_database -c "REINDEX DATABASE rg_database;"
psql -U postgres -d rg_database -c "VACUUM FULL;"

# Option 2: Restore from backup (if major corruption)
# See section 10.2 Data Loss recovery procedure

# Option 3: Failover to standby database
# See OPERATIONS_MANUAL.md section 7.2 for failover procedure
```

---

## 11. Escalation Matrix

### 11.1. Contact Information

| Role | Name | Email | Phone | Availability |
|------|------|-------|-------|--------------|
| **L1 Support** | Support Team | support@example.com | +1-XXX-XXX-XXXX | 24/7 |
| **L2 Support** | Engineering Team | eng@example.com | +1-XXX-XXX-XXXX | Business hours |
| **L3 Support / On-Call** | Senior Engineer | oncall@example.com | +1-XXX-XXX-XXXX | 24/7 |
| **DevOps Lead** | [Name] | devops@example.com | +1-XXX-XXX-XXXX | On-call rotation |
| **DBA** | [Name] | dba@example.com | +1-XXX-XXX-XXXX | On-call rotation |
| **Security Lead** | [Name] | security@example.com | +1-XXX-XXX-XXXX | 24/7 for P0 |
| **Engineering Manager** | [Name] | eng-mgr@example.com | +1-XXX-XXX-XXXX | Business hours |
| **CTO** | [Name] | cto@example.com | +1-XXX-XXX-XXXX | P0 only |

### 11.2. Escalation Path

```
L1 Support (First Response)
    ↓ (if cannot resolve in 30 min)
L2 Support / Engineering
    ↓ (if cannot resolve in 1 hour OR P0)
L3 Support / On-Call Senior Engineer
    ↓ (if P0 OR business impact > $10k)
Engineering Manager + Relevant Specialist (DevOps/DBA/Security)
    ↓ (if widespread outage OR data breach)
CTO + Executive Team
```

### 11.3. Escalation Criteria

| Severity | Initial Response | Escalate to L2 if... | Escalate to L3 if... |
|----------|------------------|----------------------|----------------------|
| **P0** | Immediate | Immediately | Immediately + Manager |
| **P1** | < 30 min | Cannot resolve in 30 min | Cannot resolve in 1 hour |
| **P2** | < 1 hour | Cannot resolve in 2 hours | Cannot resolve in 4 hours |
| **P3** | < 4 hours | Cannot resolve in 1 day | N/A |

---

## 12. Diagnostic Logs & Metrics

### 12.1. Log Locations

```bash
# Application logs
/var/log/rg-system/application.log
/var/log/rg-system/error.log
/var/log/rg-system/access.log

# Database logs
/var/log/postgresql/postgresql-14-main.log

# Redis logs
/var/log/redis/redis-server.log

# Nginx logs
/var/log/nginx/access.log
/var/log/nginx/error.log

# System logs
/var/log/syslog
/var/log/auth.log

# PM2 logs
~/.pm2/logs/rg-system-out.log
~/.pm2/logs/rg-system-error.log
```

### 12.2. Log Analysis Commands

```bash
# Find errors in last hour
find /var/log/rg-system/ -name "*.log" -mmin -60 -exec grep -i error {} \;

# Count errors by type
grep -i error /var/log/rg-system/application.log | awk '{print $5}' | sort | uniq -c | sort -rn

# Find slow requests (> 1 second)
grep "duration" /var/log/rg-system/application.log | awk '$NF > 1000 {print}' | tail -n 50

# Analyze nginx access patterns
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Check for rate limiting triggers
grep "rate limit" /var/log/nginx/access.log | wc -l
```

### 12.3. Key Metrics to Monitor

**Application Metrics:**

```promql
# Request rate
rate(http_requests_total[5m])

# Error rate
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

# Response time (p95)
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Active requests
http_requests_in_flight

# Database query duration (p95)
histogram_quantile(0.95, rate(db_query_duration_seconds_bucket[5m]))
```

**System Metrics:**

```promql
# CPU usage
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk usage
(1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100

# Network traffic
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])
```

**Database Metrics:**

```promql
# Active connections
pg_stat_activity_count

# Slow queries
pg_slow_queries_count

# Database size
pg_database_size_bytes

# Cache hit rate
pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read)
```

### 12.4. Creating Support Bundle

```bash
#!/bin/bash
# Create comprehensive support bundle for troubleshooting

BUNDLE_DIR="/tmp/rg-system-support-$(date +%Y%m%d-%H%M%S)"
mkdir -p $BUNDLE_DIR

# System info
uname -a > $BUNDLE_DIR/system-info.txt
free -h >> $BUNDLE_DIR/system-info.txt
df -h >> $BUNDLE_DIR/system-info.txt

# Application info
pm2 list > $BUNDLE_DIR/pm2-list.txt
pm2 info rg-system > $BUNDLE_DIR/pm2-info.txt
cat .env > $BUNDLE_DIR/env-config.txt  # Remove secrets before sharing!

# Logs (last 1000 lines)
tail -n 1000 /var/log/rg-system/application.log > $BUNDLE_DIR/application.log
tail -n 1000 /var/log/postgresql/postgresql-14-main.log > $BUNDLE_DIR/postgresql.log
tail -n 1000 /var/log/nginx/error.log > $BUNDLE_DIR/nginx-error.log

# Database info
psql -U rg_user -d rg_database -c "\dt+" > $BUNDLE_DIR/db-tables.txt
psql -U rg_user -d rg_database -c "SELECT version();" > $BUNDLE_DIR/db-version.txt

# Redis info
redis-cli INFO > $BUNDLE_DIR/redis-info.txt

# Metrics snapshot
curl -s http://localhost:3000/metrics > $BUNDLE_DIR/metrics.txt

# Compress bundle
tar -czf $BUNDLE_DIR.tar.gz $BUNDLE_DIR
echo "Support bundle created: $BUNDLE_DIR.tar.gz"
```

---

## Appendix A: Common Error Codes

| Error Code | Meaning | Common Cause | Resolution |
|------------|---------|--------------|------------|
| **500** | Internal Server Error | Uncaught exception, database error | Check application logs |
| **502** | Bad Gateway | Application not responding, nginx misconfiguration | Check if app is running, check nginx config |
| **503** | Service Unavailable | Application overloaded, maintenance mode | Scale up, check resource usage |
| **504** | Gateway Timeout | Request timeout (> 60s) | Optimize slow queries, increase timeout |
| **401** | Unauthorized | Invalid/expired JWT token | Re-authenticate |
| **403** | Forbidden | Insufficient permissions | Check user role/permissions |
| **404** | Not Found | Resource doesn't exist | Check URL, verify resource exists |
| **422** | Unprocessable Entity | Validation failed | Check request payload |
| **429** | Too Many Requests | Rate limit exceeded | Reduce request rate, increase rate limit |

## Appendix B: Database Error Codes

| PostgreSQL Error | Meaning | Common Cause | Resolution |
|------------------|---------|--------------|------------|
| **23505** | Unique violation | Duplicate key | Check for existing record |
| **23503** | Foreign key violation | Referenced record doesn't exist | Verify parent record exists |
| **42P01** | Undefined table | Table doesn't exist | Run migrations |
| **42703** | Undefined column | Column doesn't exist | Run migrations |
| **53300** | Too many connections | Connection pool exhausted | Increase max_connections or close idle connections |
| **57P03** | Cannot connect | Database not accepting connections | Restart PostgreSQL |
| **40P01** | Deadlock detected | Concurrent transactions conflicting | Retry transaction |

## Appendix C: Quick Reference Commands

```bash
# Health checks
curl http://localhost:3000/health
sudo systemctl status postgresql redis nginx
pm2 status

# View logs
tail -f /var/log/rg-system/application.log
sudo journalctl -u postgresql -f
pm2 logs rg-system

# Restart services
pm2 restart rg-system
sudo systemctl restart postgresql
sudo systemctl restart redis
sudo systemctl restart nginx

# Database
psql -U rg_user -d rg_database
\dt  # List tables
\d table_name  # Describe table

# Redis
redis-cli
INFO
KEYS *
GET key_name

# Monitor resources
htop
iotop
nethogs
```

---

**Document End**

**Last Updated:** 2025-11-27
**Version:** 2.13
**Maintained by:** DevOps & SRE Team
