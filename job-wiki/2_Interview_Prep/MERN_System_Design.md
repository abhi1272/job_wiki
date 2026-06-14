# MERN System Design — Senior/Principal Interview Prep

[← Back to Index](../README.md)

---

## 1. Real-Time Streaming — AI Token Response at Scale

### The Core Problem
WebSockets are stateful — each user is connected to ONE server. When you have multiple servers, the LLM response might be generated on a different server than where the user's WebSocket lives.

```
Server 1: User A, B, C connected via WebSocket
Server 2: User D, E, F connected via WebSocket
LLM Worker processes User A's request — how does it reach Server 1?
→ Answer: Redis Pub/Sub as a routing layer
```

### Full Flow
```
1. User A sends prompt → Server 1 → Kafka topic: llm.requests
2. LLM Worker consumes Kafka → calls LiteLLM with streaming
3. Each token → Redis PUBLISH "user:A:tokens" { token, done: false }
4. Server 1 (subscribed to "user:A:tokens") receives → forwards over WebSocket
5. Browser renders token immediately
6. done:true → Server 1 closes stream, Worker saves full message to MongoDB
```

### Server Code
```javascript
// One shared Redis subscriber per server (not one per user — saves connections)
const subscribers = new Map(); // userId → WebSocket
const sharedSub = redis.duplicate();
await sharedSub.pSubscribe('user:*:tokens', (message, channel) => {
  const userId = channel.split(':')[1];
  const ws = subscribers.get(userId);
  if (ws?.readyState === ws.OPEN) ws.send(message);
});

wss.on('connection', (ws, req) => {
  const userId = getUserFromToken(req);
  subscribers.set(userId, ws);

  ws.on('message', async (raw) => {
    const { prompt, conversationId, model } = JSON.parse(raw);
    await producer.send({
      topic: 'llm.requests',
      messages: [{ key: userId, value: JSON.stringify({ userId, conversationId, prompt, model }) }]
    });
  });

  ws.on('close', () => subscribers.delete(userId));
});
```

### LLM Worker Code
```javascript
await consumer.run({
  eachMessage: async ({ message }) => {
    const { userId, conversationId, prompt, model } = JSON.parse(message.value);
    const stream = await litellm.completion({ model, messages: [{ role: 'user', content: prompt }], stream: true });
    let fullResponse = '';

    for await (const chunk of stream) {
      const token = chunk.choices[0]?.delta?.content || '';
      fullResponse += token;
      await redis.publish(`user:${userId}:tokens`, JSON.stringify({ token, done: false }));
    }

    await redis.publish(`user:${userId}:tokens`, JSON.stringify({ token: '', done: true }));
    await Message.create({ conversationId, role: 'assistant', content: fullResponse });
  }
});
```

### Why Kafka + Redis (not just one)
```
Kafka  → durable, replayable, load-balanced LLM request queue (client → worker)
Redis  → ephemeral, instant pub/sub routing (worker → correct server → client)
MongoDB→ persistent message storage after stream completes
```

### Architecture
```
Browser ─WebSocket─► Load Balancer
                          ├── Server 1 ─┐
                          ├── Server 2 ─┼── Redis Pub/Sub ◄── LLM Workers ◄── Kafka ◄── Servers
                          └── Server 3 ─┘                           │
                                                                MongoDB (persist)
```

### Interview Answer
> "The challenge is routing LLM responses to the right user across multiple servers. Solution: Redis Pub/Sub as a routing layer. User sends prompt → Kafka for durable LLM processing. Worker streams tokens → publishes to Redis channel keyed by userId. Every server subscribes to its users' channels — whichever holds the WebSocket forwards immediately. For 10k users I use one shared Redis subscriber per server with an in-memory Map routing to the correct WebSocket — avoiding 10k Redis connections."

---

---

## 2. Authentication — JWT vs Sessions at Scale

### JWT — Stateless, Can't Be Revoked
```javascript
const accessToken = jwt.sign({ userId, orgId, role }, secret, { expiresIn: '15m' });
jwt.verify(token, secret); // no DB/Redis lookup — just math
```
**Pros:** Stateless (any server verifies), works for mobile/M2M, carries claims  
**Critical flaw:** Can't revoke before expiry — logout doesn't truly log out

### Sessions — Stateful, Instant Revocation
```javascript
app.use(session({
  store: new RedisStore({ client: redisClient }), // survives restarts — fixes your concern
  cookie: { secure: true, httpOnly: true, sameSite: 'strict' }
}));
// Delete session from Redis → user logged out everywhere immediately
```
**Pros:** Instant revocation, httpOnly cookie blocks XSS, easy "logout all devices"  
**Con:** Requires Redis, every request hits Redis (~1ms), browser-centric

### Production Answer — Hybrid (Best of Both)
```javascript
// Login: issue short JWT + long opaque refresh token
const accessToken  = jwt.sign({ userId, orgId }, secret, { expiresIn: '15m' });
const refreshToken = crypto.randomBytes(64).toString('hex');
await redis.setEx(`refresh:${refreshToken}`, 7*24*60*60, JSON.stringify({ userId }));
// refreshToken → httpOnly cookie, accessToken → response body

// API calls: verify JWT — no Redis hit
const payload = jwt.verify(req.headers.authorization.split(' ')[1], secret);

// Access token expired: use refresh token → get new access token
const data = await redis.get(`refresh:${req.cookies.refreshToken}`);
if (!data) throw new UnauthorizedError();
const newAccessToken = jwt.sign(JSON.parse(data), secret, { expiresIn: '15m' });

// Logout: delete refresh token from Redis — instant revocation ✅
await redis.del(`refresh:${req.cookies.refreshToken}`);
```

### JWT Revocation Options
```
Short expiry (15min)       → stolen token valid for max 15 min — acceptable for most
Redis blocklist            → store jti of revoked tokens — makes JWT stateful again
Token version in DB        → increment user.tokenVersion → all old JWTs invalid
                             requires DB lookup per request
```

### When to Use What
```
Pure JWT         → M2M auth, microservices, mobile, simple APIs
Sessions + Redis → high security web apps, need instant revocation
Hybrid           → production web apps at scale — best of both
```

### Interview Answer
> "JWT is stateless and scales without shared storage, but the critical flaw is you can't revoke before expiry — a stolen token stays valid. Sessions solve this but need Redis to survive server restarts. For 50k users I'd use hybrid: short-lived JWTs (15 min) for stateless API calls, opaque refresh tokens in Redis as httpOnly cookies. Access tokens expire fast enough that theft window is acceptable. Refresh tokens can be instantly revoked by deleting from Redis. httpOnly prevents XSS theft."

---

---

## 3. File Upload Architecture — Large Files (up to 500MB)

### Never proxy through Node.js
Loading 500MB into Node.js heap crashes the process. Client uploads directly to S3.

### Pre-signed URL Flow
```
Client → POST /upload-url (filename, size, type) → validation → S3 pre-signed URL
Client → PUT directly to S3 (bypasses Node.js entirely, with browser progress %)
Client → POST /confirm → Node.js marks processing + publishes to Kafka
Kafka → KB Worker → S3 stream → extract → chunk → embed → vector DB
WebSocket → progress updates back to client (same Redis Pub/Sub pattern)
```

### Server Code
```javascript
// Step 1 — issue pre-signed URL with validation
app.post('/api/files/upload-url', asyncHandler(async (req, res) => {
  const { filename, contentType, size } = req.body;
  if (size > 500 * 1024 * 1024) throw new ValidationError('Max 500MB');
  if (!['application/pdf', 'application/vnd.openxmlformats-...'].includes(contentType))
    throw new ValidationError('Only PDF and DOCX');

  const fileId = `${req.user.orgId}/${uuid()}-${filename}`;
  const uploadUrl = await s3.getSignedUrlPromise('putObject', {
    Bucket: S3_BUCKET, Key: fileId, ContentType: contentType, Expires: 900
  });

  await KBDocument.create({ fileId, orgId: req.user.orgId, status: 'upload_pending' });
  res.json({ uploadUrl, fileId });
}));

// Step 2 — confirm upload, queue processing
app.post('/api/files/:fileId/confirm', asyncHandler(async (req, res) => {
  await s3.headObject({ Bucket: S3_BUCKET, Key: req.params.fileId }).promise(); // verify exists
  await KBDocument.updateOne({ fileId: req.params.fileId }, { status: 'processing' });
  await producer.send({ topic: 'kb.process', messages: [{ value: JSON.stringify({ fileId: req.params.fileId, orgId: req.user.orgId }) }] });
  res.json({ status: 'processing' });
}));
```

### KB Worker Pipeline
```javascript
// 1. Download from S3 as STREAM (not into memory)
const s3Stream = s3.getObject({ Bucket: S3_BUCKET, Key: fileId }).createReadStream();

// 2. Extract text (pdf-parse for PDF, mammoth for DOCX)
const text = await extractText(s3Stream, contentType);

// 3. Chunk with overlap for context continuity
const chunks = chunkText(text, { chunkSize: 512, overlap: 50 });

// 4. Embed in batches (respect API rate limits)
for (const batch of chunk(chunks, 100)) {
  const embeddings = await openai.embeddings.create({ model: 'text-embedding-3-small', input: batch.map(c => c.text) });
  // store embeddings...
}

// 5. Upsert to vector DB
await vectorDB.upsert({ namespace: `${orgId}_kb`, vectors: chunks.map((c, i) => ({
  id: `${fileId}_chunk_${i}`, values: embeddings[i], metadata: { text: c.text, fileId }
})) });

// 6. Progress via Redis → WebSocket
await redis.publish(`kb:${fileId}:progress`, JSON.stringify({ step: 'complete', pct: 100 }));

// Retry + DLQ on failure
// retryCount < 3 → kb.process.retry topic
// retryCount >= 3 → kb.process.dlq (dead letter queue)
```

### Interview Answer
> "For 500MB files, client never uploads through Node.js — server generates a pre-signed S3 URL with size and type validation. Client uploads directly to S3 with browser upload progress, then confirms to the API which queues a Kafka job. The async KB worker downloads as a stream, extracts text, chunks with overlap for context, embeds in batches to respect rate limits, upserts to vector DB. Progress flows back via Redis Pub/Sub → WebSocket. Failed jobs retry 3 times then go to a dead letter queue."

---

---

## 4. Caching Strategy

### Three Questions Per Data Type
```
1. How often does it change?   → TTL
2. How bad is stale data?      → invalidation approach
3. How expensive is the query? → worth caching?
```

### Cache Strategy Per Endpoint
```
Data                  TTL       Invalidation               Why
Agent config          1 hour    Event-driven on update     Read every request, changes rarely
Available models      24 hours  Manual on model change     Changes ~monthly
Usage stats           5 min     TTL only                   Staleness OK, avoids expensive aggregation
Recent conversations  30 sec    Event-driven on new msg    Needs to feel fresh-ish
Billing summary       1 min     TTL only                   1 min stale acceptable
```

### Cache-Aside Pattern (standard)
```javascript
async function getAgent(agentId, orgId) {
  const key = `agent:${orgId}:${agentId}`;
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const agent = await Agent.findOne({ _id: agentId, orgId });
  await redis.setEx(key, 3600, JSON.stringify(agent)); // 1 hour
  return agent;
}

// Event-driven invalidation — don't wait for TTL
async function updateAgent(agentId, updates) {
  const agent = await Agent.findOneAndUpdate({ _id: agentId }, updates, { new: true });
  await redis.del(`agent:*:${agentId}`); // bust immediately ✅
  return agent;
}
```

### Multi-Layer Caching (high-traffic endpoints)
```javascript
const localCache = new NodeCache({ stdTTL: 30 }); // L1: in-process (0ms)

async function getAgentFast(agentId, orgId) {
  const key = `agent:${orgId}:${agentId}`;

  const local = localCache.get(key);           // L1: 0ms
  if (local) return local;

  const redisVal = await redis.get(key);       // L2: ~1ms
  if (redisVal) {
    const agent = JSON.parse(redisVal);
    localCache.set(key, agent);
    return agent;
  }

  const agent = await Agent.findOne({ _id: agentId, orgId }); // L3: 5-50ms
  await redis.setEx(key, 3600, JSON.stringify(agent));
  localCache.set(key, agent);
  return agent;
}
// Trade-off: L1 is per-server — 30 sec max staleness after update on other servers
```

### Cache Stampede Prevention (dog-pile problem)
1000 requests hit at once, cache empty → 1000 simultaneous DB queries → DB crash.

```javascript
async function getWithLock(key, fetchFn, ttl) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const lock = await redis.set(`lock:${key}`, '1', { NX: true, EX: 5 }); // only one gets lock
  if (!lock) {
    await new Promise(r => setTimeout(r, 100)); // wait for lock holder to populate
    return getWithLock(key, fetchFn, ttl);       // retry — will hit cache this time
  }

  try {
    const value = await fetchFn();
    await redis.setEx(key, ttl, JSON.stringify(value));
    return value;
  } finally {
    await redis.del(`lock:${key}`);
  }
}
```

### Interview Answer
> "Each data type needs its own TTL and invalidation strategy. Agent configs: 1-hour Redis cache, event-driven invalidation on update. Models: 24-hour TTL, manual bust. Stats: 5-minute TTL only — staleness is acceptable, avoids expensive aggregation on every load. For high-traffic: two-layer caching — in-process NodeCache (0ms) backed by Redis. The cache stampede problem — expired cache + 1000 simultaneous requests — is solved with a Redis distributed lock: first request fetches and populates, others wait and retry."

---

## 5. Multi-Tenant SaaS Architecture

### Three Tenancy Models
```
Shared (Pool)     → one DB, orgId on every doc — cheapest, standard clients
Separate Schema   → one DB, separate namespace per tenant — middle ground  
Silo              → dedicated DB per tenant — max isolation, enterprise/regulated
```

### Dynamic DB Switching
```javascript
const connections = new Map(); // orgId → mongoose.Connection

async function getTenantDB(orgId) {
  const config = await getTenantConfig(orgId); // from Redis cache
  if (config.tier === 'standard') return mongoose.connection; // shared

  if (connections.has(orgId)) return connections.get(orgId);  // reuse existing

  const conn = await mongoose.createConnection(config.dbUri, { maxPoolSize: 10 });
  connections.set(orgId, conn);
  return conn;
}

// Middleware — resolves tenant on every request
app.use(async (req, res, next) => {
  const orgId = req.user?.orgId;
  req.tenant = {
    orgId,
    config: await getTenantConfig(orgId), // Redis cached — no DB hit
    db: await getTenantDB(orgId)
  };
  next();
});

// Route — uses tenant's DB, not global mongoose
app.get('/api/agents', asyncHandler(async (req, res) => {
  const Agent = req.tenant.db.model('Agent');
  const agents = await Agent.find({ orgId: req.tenant.orgId });
  res.json(agents);
}));
```

### Per-Tenant Encryption (AWS KMS)
```javascript
async function encryptField(orgId, plaintext) {
  const { encryptionKeyArn } = await getTenantConfig(orgId);
  const { CiphertextBlob } = await kms.send(new EncryptCommand({
    KeyId: encryptionKeyArn || process.env.DEFAULT_KMS_KEY,
    Plaintext: Buffer.from(plaintext)
  }));
  return CiphertextBlob.toString('base64');
}
// Bank's data encrypted with THEIR KMS key — unreadable even to you in a breach
```

### Defence in Depth — Preventing Cross-Tenant Leaks
```javascript
// Base repository — orgId ALWAYS injected into every query
class BaseRepository {
  constructor(model, orgId) { this.model = model; this.orgId = orgId; }

  find(filter = {}) {
    return this.model.find({ ...filter, orgId: this.orgId }); // can't forget orgId
  }
  findById(id) {
    return this.model.findOne({ _id: id, orgId: this.orgId }); // cross-tenant → null
  }
}
const agentRepo = new BaseRepository(Agent, req.tenant.orgId);
```

### Two-Tier Architecture
```
Standard clients → Shared MongoDB + Shared Redis + orgId isolation
Enterprise (bank) → Dedicated MongoDB cluster + Dedicated Redis + Own KMS key + Dedicated S3
Both → same API, same codebase, tenant middleware handles routing
```

### Interview Answer
> "Three tenancy models: shared database with orgId isolation for standard clients, dedicated database silo for regulated enterprise clients. For the bank: dedicated MongoDB cluster, their own KMS encryption key — their data is unreadable even in a breach. Code switches databases dynamically via a connection pool keyed by orgId; tenant config is Redis-cached so there's no DB hit per request. Defence against cross-tenant leaks: base repository layer always injects orgId into every query — developers physically can't write a query that crosses tenant boundaries."
