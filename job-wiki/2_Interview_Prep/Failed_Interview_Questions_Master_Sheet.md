# ❌ → ✅ Failed Interview Questions — Master Answer Sheet
> Questions I was asked and failed to answer properly.
> Study these before every interview. Say each answer out loud.

---

## 📌 HOW TO USE THIS FILE
1. Cover the answer section
2. Try to answer from memory
3. Uncover and compare
4. If you got it wrong — mark it with ❌ and revisit next session
5. If you got it right 3 times in a row — mark ✅ and move on

---

## QUESTION 1 — Why Redis Over In-Memory (Node.js)?

### ❌ What I Said
*Vague answer — didn't explain the distributed systems problem clearly*

### ✅ What I Should Have Said

**The core problem with in-memory:**

```
Node.js Pod 1  →  memory: { EMP-123: IN  }
Node.js Pod 2  →  memory: { EMP-123: OUT }

Load balancer sends requests to different pods
→ Two pods, two different answers for same data
→ Inconsistent state across instances
```

**4 specific problems with in-memory:**
1. **No shared state** — 10 pods = 10 separate memories, data is inconsistent
2. **Lost on restart** — pod crashes → idempotency keys gone → duplicate events processed
3. **Memory pressure** — app memory used for both requests AND data = GC pressure = latency spikes
4. **No TTL** — you write your own cleanup logic, error-prone

**Redis solves all four:**
```
All pods → single Redis → consistent shared state
Persists on restart (RDB/AOF)
Built-in TTL per key — SET key value EX 3600
Sub-millisecond at 100k+ ops/sec
```

**One-liner to memorise:**
> *"In-memory breaks the moment you have more than one server instance. Redis is the shared memory layer for your entire distributed system — consistent, persistent, and fast."*

---

## QUESTION 2 — On Which Column Should You NOT Place an Index?

### ❌ What I Said
*Couldn't give clear examples with reasoning*

### ✅ What I Should Have Said

**Never index these — with reason:**

| Column Type | Example | Why Not |
|---|---|---|
| Low cardinality | `gender`, `status`, `boolean` | 2-3 values = index covers 50% of rows = DB ignores it |
| Rarely queried | `middle_name`, `notes` | Never used in WHERE = index maintained for nothing |
| Frequently updated | `last_seen_at`, `updated_at` | Every update rebuilds the index entry = write slowdown |
| Small tables | Countries (500 rows) | Full scan of 500 rows is faster than index lookup overhead |
| Already covered | `org_id` when you have `(org_id, employee_id)` | Composite index leftmost column already covers this |
| Function results in reports | `EXTRACT(YEAR FROM hire_date)` | Runs once a year, slows every INSERT for 364 days |

**The rule:**
```
Index WHEN:  column in WHERE, JOIN ON, ORDER BY — frequently
Skip WHEN:   low cardinality, rarely queried, frequently updated,
             small table, already covered by composite index
```

**Low cardinality explained:**
```sql
-- BAD: status has only 2 values (active/inactive)
CREATE INDEX idx_status ON employees(status);
-- Index points to 50% of table
-- PostgreSQL decides full scan is faster
-- Index never used, but every INSERT pays the cost
```

---

## QUESTION 3 — How to Avoid Upload of Malicious Files?

### ❌ What I Said
*Only mentioned file extension check — weakest defence*

### ✅ What I Should Have Said

**5 layers of defence — never rely on just one:**

**Layer 1 — File extension check (weakest — easily faked)**
```javascript
const allowedExtensions = ['.jpg', '.png', '.pdf'];
const ext = path.extname(file.originalname).toLowerCase();
if (!allowedExtensions.includes(ext)) throw new Error('Invalid type');
```

**Layer 2 — Magic bytes check (real validation — can't be faked)**
```javascript
// Every file format has a unique byte signature at the start
// JPEG: FF D8 FF | PNG: 89 50 4E 47 | PDF: 25 50 44 46
const { fileTypeFromBuffer } = require('file-type');
const type = await fileTypeFromBuffer(file.buffer.slice(0, 8));
if (!allowedMimes.includes(type.mime)) throw new Error('Content mismatch');
// File claiming to be .jpg but magic bytes say .exe → BLOCKED
```

**Layer 3 — Size limits (multiple levels)**
```javascript
const upload = multer({ limits: { fileSize: 10 * 1024 * 1024, files: 1 } });
// Also set at Nginx: client_max_body_size 10m;
```

**Layer 4 — Antivirus scan before storing**
```javascript
const { isInfected, viruses } = await clamscan.scanBuffer(file.buffer);
if (isInfected) throw new Error('File failed security scan');
// Scan BEFORE writing to storage — never store first, scan later
```

**Layer 5 — Safe private storage + rename**
```javascript
// NEVER store in web-accessible directory
// BAD: /var/www/public/uploads/malware.php → browser executes it
// GOOD: Private S3 bucket, served via pre-signed URLs only
const safeFilename = `${uuid()}.${allowedExtension}`;
// Strips original filename — prevents path traversal attacks
```

**Bonus for images — re-encode through Sharp:**
```javascript
const safeImage = await sharp(file.buffer).jpeg({ quality: 85 }).toBuffer();
// Re-encoding strips all embedded payloads and EXIF exploits
```

**One-liner:**
> *"Extension check is the weakest defence — it's easily faked. Magic bytes validation, antivirus scanning, and private S3 storage with renamed files are the real defences. Never store first and scan later."*

---

## QUESTION 4 — How to Check API Security During Development?

### ❌ What I Said
*Only mentioned manual testing — missed tooling*

### ✅ What I Should Have Said

**5 tools — shift security left into development:**

**1. OWASP ZAP (automated scanner)**
```bash
zap-baseline.py -t http://localhost:3000/api
# Finds: SQL injection, XSS, broken auth, sensitive data exposure
# Integrate into CI/CD — fails build if HIGH severity found
```

**2. JWT edge case tests in Postman/Jest**
```bash
# Test expired token → must return 401
# Test tampered payload → must return 401
# Test missing token → must return 401
# Test token from different environment → must return 401
# Test accessing another user's resource → must return 403
```

**3. ESLint Security Plugin (catches issues on save)**
```javascript
// .eslintrc — catches eval(), path traversal, regex DoS
{ "plugins": ["security"], "extends": ["plugin:security/recommended"] }
```

**4. npm audit + Snyk (dependency vulnerabilities)**
```bash
npm audit --audit-level=high
snyk test  # deeper scanning with fix suggestions
```

**5. Secret detection (prevent credentials in git)**
```bash
git secrets --scan  # scans before commit, blocks if API keys found
```

**In CI/CD pipeline:**
```yaml
security-scan:
  script:
    - npm audit --audit-level=high
    - snyk test
    - zap-baseline.py -t $API_URL
  allow_failure: false  # Block merge on security issues
```

---

## QUESTION 5 — How to Stop XSS?

### ❌ What I Said
*Mentioned sanitisation but couldn't explain the 3 types or CSP*

### ✅ What I Should Have Said

**3 types of XSS and how to stop each:**

**Type 1 — Stored XSS (attacker saves malicious script in DB)**
```javascript
// Attacker saves name = <script>steal(document.cookie)</script>
// Every user who views this page → script executes

// Fix: Output encoding
const he = require('he');
res.send(`<p>${he.encode(user.name)}</p>`);
// Renders as literal text — never executes
// React does this automatically: <p>{user.name}</p> is safe
```

**Type 2 — Reflected XSS (malicious URL reflects into page)**
```javascript
// BAD: reflects raw query param into HTML
res.send(`<p>Results for: ${req.query.q}</p>`);

// GOOD:
res.send(`<p>Results for: ${he.encode(req.query.q)}</p>`);
```

**Type 3 — DOM XSS (client-side JS writes to DOM)**
```javascript
// BAD: interprets HTML
document.getElementById('output').innerHTML = userInput;

// GOOD: never interprets HTML
document.getElementById('output').textContent = userInput;
```

**Content Security Policy — the safety net:**
```javascript
// Even if XSS gets through — CSP blocks execution
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"],   // Only scripts from your domain
    objectSrc: ["'none'"]    // No Flash/plugins
  }
}));
// Browser refuses to run injected scripts from other domains
```

**One-liner:**
> *"Use textContent not innerHTML, encode all output with a library like he.js, and add Content Security Policy headers via Helmet as a safety net — even if encoding is missed somewhere, CSP blocks execution."*

---

## QUESTION 6 — What Can WAF Handle?

### ❌ What I Said
*Only said "blocks attacks" — too vague*

### ✅ What I Should Have Said

**WAF sits between internet and your app — inspects every HTTP request:**

```
Internet → WAF (inspects URL, headers, body, cookies) → Your App
```

**What WAF handles:**
```
✅ XSS — detects script injection patterns in requests
✅ SQL Injection — detects SQL patterns in parameters
✅ OWASP Top 10 attack patterns
✅ DDoS / Rate limiting — blocks IP flooding
✅ Bot detection — scraping, credential stuffing attacks
✅ Geo-blocking — block requests from specific countries
✅ IP reputation — blocks known malicious IPs automatically
✅ Request size limits — prevents large payload attacks
✅ HTTP method enforcement — block DELETE if not needed
✅ Virtual patching — block known CVEs before you patch code
```

**WAF options:**
| WAF | Best For |
|---|---|
| AWS WAF | AWS — integrates with ALB, CloudFront |
| Cloudflare WAF | Any stack — also CDN + DDoS protection |
| Cloud Armor | GCP — integrates with Load Balancer |
| ModSecurity | Self-hosted, open source |

**Critical distinction — WAF is first line, not only line:**
```
WAF:
✅ Blocks known attack patterns at network level
✅ Zero code changes needed
❌ Can be bypassed with obfuscation
❌ Can't understand your business logic

Application security:
✅ Understands business context
✅ Can't be bypassed by obfuscation
❌ Requires developer discipline

Answer: Both — WAF + application security = defence in depth
```

---

## QUESTION 7 — How Does CDN Work?

### ❌ What I Said
*Explained basics but missed cache invalidation and Cache-Control headers*

### ✅ What I Should Have Said

**Core concept:**
```
Without CDN:
User in Mumbai → Your server in US → ~200ms latency

With CDN:
User in Mumbai → CDN edge node in Mumbai → ~5ms latency

First request (cache miss): Mumbai edge → fetches from origin → caches
Subsequent requests: Mumbai edge → serves from cache → origin never called
```

**Cache-Control headers control CDN behaviour:**
```
Cache-Control: public, max-age=31536000, immutable
→ Cache 1 year — for JS/CSS/images with content hash in filename

Cache-Control: public, max-age=300
→ Cache 5 minutes — for semi-static API responses

Cache-Control: private, no-store
→ Never cache — for authenticated/personal data responses
```

**Cache invalidation — the hard problem:**
```
Solution 1 — Content hashing (best practice):
app.js → app.a3b4c5d6.js (hash of file content)
New deploy → app.f7g8h9i0.js (new hash = new filename)
HTML references new name → CDN fetches fresh automatically
Old name never requested again → no stale content

Solution 2 — CDN purge API:
Cloudflare/CloudFront API → purge /static/* on deploy
Triggered by deployment pipeline

Solution 3 — Short TTL:
Cache-Control: max-age=300 → stale max 5 minutes
Simpler but origin gets more traffic
```

**What CDN handles beyond caching:**
```
✅ DDoS protection — Cloudflare absorbs attack before origin
✅ SSL termination — HTTPS at edge, HTTP internally
✅ Compression — Gzip/Brotli at edge reduces transfer size
✅ Image optimisation — WebP conversion, resize on the fly
✅ Geographic routing — nearest healthy edge node
```

---

## QUESTION 8 — Plain SQL vs ORM?

### ❌ What I Said
*Said ORM is better without explaining when to use each*

### ✅ What I Should Have Said

**Use both — different tools for different jobs:**

**ORM (Prisma, TypeORM, Sequelize) — use for:**
```
✅ Simple CRUD operations
✅ Standard reads by primary key or indexed column
✅ Authentication flows
✅ Basic list/filter APIs
✅ Anything under 3 table joins
✅ When type safety matters (TypeScript + Prisma = excellent DX)
✅ Database migrations as code
```

**Plain SQL — use for:**
```
✅ Complex reports and aggregations (SUM, COUNT, AVG, GROUP BY)
✅ Payroll calculations (window functions, CTEs)
✅ Bulk operations (INSERT ... SELECT, bulk updates)
✅ Analytics queries across large datasets
✅ Performance-critical paths where you need exact query control
✅ When EXPLAIN ANALYZE shows ORM generating bad query
```

**ORM pitfall — N+1 problem:**
```javascript
// BAD — ORM doing N+1 without you realising
const employees = await Employee.findAll();
for (const emp of employees) {
  emp.shifts = await Shift.findAll({ where: { employeeId: emp.id }});
}
// 1 query for employees + N queries for shifts = N+1

// GOOD — eager loading (single JOIN query)
const employees = await Employee.findAll({ include: [{ model: Shift }] });
```

**Plain SQL pitfall — SQL injection if not careful:**
```javascript
// BAD — string concatenation = SQL injection
const result = await pool.query(`SELECT * FROM employees WHERE id = ${userId}`);

// GOOD — parameterised query always
const result = await pool.query('SELECT * FROM employees WHERE id = $1', [userId]);
```

**One-liner:**
> *"ORM for developer productivity on standard CRUD, raw SQL for complex aggregations and performance-critical paths. Always run EXPLAIN ANALYZE on ORM-generated queries — if they're inefficient, drop to raw SQL for that specific query."*

---

## QUESTION 9 — How to Trace a Request From Start to End?

### ❌ What I Said
*Mentioned logs but didn't know about distributed tracing or TraceId propagation*

### ✅ What I Should Have Said

**The problem:**
```
User reports: "My check-in at 9:15 AM wasn't recorded"
Without tracing: grep millions of log lines across 5 services
With tracing: one TraceId query → full picture in seconds
```

**How distributed tracing works:**
```
Request arrives → Generate TraceId: "abc-123-def-456"

API Gateway        → creates SpanId: span-001, logs with TraceId
  ↓ passes TraceId in header
Attendance Service → creates SpanId: span-002, logs with TraceId
  ↓ publishes to Kafka WITH TraceId in message headers
Kafka Consumer     → extracts TraceId, creates SpanId: span-003
  ↓
PostgreSQL write   → tagged with TraceId

Query TraceId "abc-123-def-456" → see entire journey
```

**Implementation with OpenTelemetry:**
```javascript
// Setup once — auto-instruments HTTP, Express, PostgreSQL, Redis, Kafka
const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: 'http://jaeger:4318' }),
  instrumentations: [getNodeAutoInstrumentations()]
});
sdk.start();
// No other code changes needed for basic tracing
```

**Propagate TraceId through Kafka:**
```javascript
// Producer — inject trace context into message headers
const carrier = {};
propagation.inject(context.active(), carrier);
await producer.send({
  topic: 'attendance.raw.events',
  messages: [{ value: JSON.stringify(event), headers: carrier }]
});

// Consumer — extract and continue the trace
const parentContext = propagation.extract(context.active(), message.headers);
context.with(parentContext, async () => {
  await processEvent(message.value); // child of original trace
});
```

**What you see in Jaeger/Grafana Tempo:**
```
TraceId: abc-123-def-456                           [28ms total]
  ├── API Gateway: authenticate-jwt                [2ms]
  ├── Attendance Service: validate-geofence        [3ms]
  ├── Redis: check-idempotency                     [1ms]
  ├── Kafka: publish event                         [1ms]
  └── Kafka Consumer: process event                [21ms]
        ├── Redis: fetch shift (cache miss!)       [15ms] ← SLOW
        ├── PostgreSQL: fetch shift from DB        [4ms]
        └── PostgreSQL: write attendance           [2ms]
```

**Instantly see:** Redis cache miss caused the slowness. Fix the cache.

**Correlation in logs — every log line gets TraceId:**
```javascript
logger.info('Check-in processed', {
  employeeId: event.employeeId,
  traceId: trace.getActiveSpan()?.spanContext().traceId
});
// Search logs by traceId → all lines from all services for one request
```

**Tools:**
- **Jaeger** — open source, self-hosted
- **Grafana Tempo** — open source, integrates with Grafana dashboards
- **Datadog APM** — paid, best UI
- **AWS X-Ray** — if on AWS

**One-liner:**
> *"OpenTelemetry generates a TraceId at the entry point and propagates it through every service call and Kafka message. One TraceId query in Jaeger shows the complete request journey — which service was slow, which DB query failed, what happened in Kafka — across your entire distributed system."*

---

## 📊 SELF ASSESSMENT TRACKER

Rate yourself after each revision session:

| Question | Session 1 | Session 2 | Session 3 | Ready? |
|---|---|---|---|---|
| Redis vs In-Memory | ⬜ | ⬜ | ⬜ | ⬜ |
| When NOT to index | ⬜ | ⬜ | ⬜ | ⬜ |
| Malicious file upload | ⬜ | ⬜ | ⬜ | ⬜ |
| API security dev | ⬜ | ⬜ | ⬜ | ⬜ |
| Stop XSS | ⬜ | ⬜ | ⬜ | ⬜ |
| WAF capabilities | ⬜ | ⬜ | ⬜ | ⬜ |
| CDN + cache invalidation | ⬜ | ⬜ | ⬜ | ⬜ |
| SQL vs ORM | ⬜ | ⬜ | ⬜ | ⬜ |
| Distributed tracing | ⬜ | ⬜ | ⬜ | ⬜ |

**Rating scale:** ❌ Blank = couldn't answer | 🟡 Partial = got some | ✅ Full = nailed it

---

## ⚡ QUICK RECALL — ONE-LINERS FOR EACH

```
REDIS vs IN-MEMORY:
  "In-memory breaks with multiple instances.
   Redis = shared memory for distributed systems."

NO INDEX ON:
  "Low cardinality, rarely queried, frequently updated,
   small tables, covered by composite index."

MALICIOUS UPLOAD:
  "Magic bytes not extension. Antivirus before storing.
   Private S3, renamed files. Re-encode images through Sharp."

API SECURITY:
  "OWASP ZAP + ESLint security plugin + JWT edge cases
   + npm audit + Snyk in CI/CD."

STOP XSS:
  "textContent not innerHTML. Output encoding with he.js.
   Content Security Policy via Helmet as safety net."

WAF:
  "First line not only line. OWASP Top 10, DDoS, bots,
   geo-blocking at network layer. App security still needed."

CDN:
  "Edge nodes cache near users. Cache-Control headers control
   behaviour. Content hashing solves cache invalidation."

SQL vs ORM:
  "ORM for CRUD, raw SQL for aggregations and reports.
   EXPLAIN ANALYZE ORM queries — drop to SQL if inefficient."

DISTRIBUTED TRACING:
  "OpenTelemetry TraceId propagated through every service
   and Kafka message. One query = full request journey."
```

---

## 📅 REVISION SCHEDULE

| When | What to do |
|---|---|
| Today | Read all answers once, say out loud |
| Tomorrow morning | Cover answers, try to recall each one |
| Day 3 | Self-assessment tracker — mark each |
| Day 5 | Focus only on questions still marked ❌ or 🟡 |
| Night before interview | Quick-recall one-liners only |

---

*Every question here was a real gap. Fix them once — own them forever.*
