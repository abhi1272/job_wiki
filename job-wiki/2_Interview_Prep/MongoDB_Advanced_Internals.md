# MongoDB Advanced Internals — MERN Senior/Principal Interview Prep

[← Back to Index](../README.md)

---

## 1. Schema Design — Embed vs Reference

### The Golden Rule
> Design your schema around your **access patterns**, not your relationships.

### Decision Framework
```
EMBED when:
  ✅ Data always accessed together with parent
  ✅ Data owned by one parent (not shared)
  ✅ Small, bounded size (< ~20 items)
  ✅ One-to-few relationship

REFERENCE when:
  ✅ Data accessed independently of parent
  ✅ Data shared across multiple parents
  ✅ Can grow unboundedly
  ✅ One-to-many or many-to-many
```

### Agent Builder Schema

**agents** — tools EMBEDDED (always loaded together, bounded, owned by one agent)
```javascript
{
  _id: ObjectId("agent123"),
  orgId: "org_abc",              // tenant isolation — first in all indexes
  name: "Customer Support Bot",
  systemPrompt: "You are helpful...",
  model: "claude-3-sonnet",
  tools: [                       // EMBED — small list, always needed with agent
    { name: "search_web", enabled: true, config: { maxResults: 5 } },
    { name: "query_db",   enabled: true, config: { database: "crm" } }
  ],
  kbIds: ["kb_001", "kb_002"],  // REFERENCE — KBs shared across agents, fetched by RAG separately
  status: "active",
  createdAt: ISODate("2024-01-15")
}
```

**knowledge_bases** — SEPARATE collection (shared, heavy, fetched independently)
```javascript
{
  _id: ObjectId("kb_001"),
  orgId: "org_abc",
  name: "Product Documentation",
  documentCount: 145,
  vectorNamespace: "org_abc_kb_001",  // pointer to Pinecone/pgvector
  status: "indexed"
  // actual vectors live in vector DB — not here
}
```

**conversations** — HYBRID pattern
```javascript
{
  _id: ObjectId("conv_789"),
  orgId: "org_abc",
  agentId: "agent123",          // REFERENCE — not embedded (agent changes independently)
  userId: "user_xyz",
  lastMessageAt: ISODate("2024-01-20T15:30:00"),
  messageCount: 24,
  recentMessages: [             // EMBED last 10 — fast "resume chat" load
    { role: "user", content: "How do I reset my password?", timestamp: ISODate(...) },
    { role: "assistant", content: "You can reset...", tokens: { in: 120, out: 85 } }
  ]
}
```

**messages** — SEPARATE collection (unbounded growth, 16MB doc limit if embedded)
```javascript
{
  _id: ObjectId("msg_001"),
  orgId: "org_abc",
  conversationId: "conv_789",
  role: "user",                  // user | assistant | tool
  content: "How do I reset my password?",
  timestamp: ISODate("2024-01-20T15:29:00"),
  toolCall: { name: "search_kb", input: {}, output: {}, durationMs: 234 },
  usage: { inputTokens: 120, outputTokens: 85, cost: 0.00043 }
}
```

### Indexes — Multi-Tenant Critical
```javascript
// orgId FIRST in all compound indexes — filters tenant before anything else
db.agents.createIndex({ orgId: 1, status: 1 })
db.agents.createIndex({ orgId: 1, createdAt: -1 })

db.conversations.createIndex({ orgId: 1, agentId: 1, lastMessageAt: -1 })
db.conversations.createIndex({ lastMessageAt: 1 }, { expireAfterSeconds: 7776000 }) // TTL 90d

db.messages.createIndex({ conversationId: 1, timestamp: 1 })  // paginated history
db.messages.createIndex({ orgId: 1, agentId: 1, timestamp: -1 }) // analytics
```

### Pre-Aggregated Stats (avoid aggregating millions of messages)
```javascript
// agent_stats — updated on every message, instant dashboard queries
{
  _id: "agent123_2024_01",
  orgId: "org_abc",
  agentId: "agent123",
  period: "2024-01",
  totalMessages: 8930,
  totalInputTokens: 1230000,
  totalCost: 12.45
}
db.agent_stats.updateOne(
  { _id: `${agentId}_${yearMonth}` },
  { $inc: { totalMessages: 1, totalInputTokens: usage.input } },
  { upsert: true }
);
```

### Interview Answer
> "MongoDB schema design starts with access patterns. Tools are embedded in the agent — always loaded together, bounded, owned by one agent. KBs are referenced — shared across agents, fetched independently by the RAG pipeline. Messages are a separate collection — unbounded growth hits the 16MB document limit if embedded. I use a hybrid: embed last 10 messages in conversation for fast resume-chat, full history fetched from messages with pagination. orgId is always first in compound indexes for multi-tenant isolation. Pre-aggregated stats documents avoid expensive aggregations over millions of message records."

---

---

## 2. Aggregation Pipeline

### Mental Model
Data flows through stages like a Unix pipe:
```
Collection → [$match] → [$group] → [$lookup] → [$project] → [$sort] → result
```

### MongoDB HAS Joins — `$lookup`
```javascript
// $lookup = LEFT OUTER JOIN
{ $lookup: { from: "agents", localField: "_id", foreignField: "_id", as: "agentInfo" } }
// Avoid at scale — use pre-aggregated stats instead
```

### Monthly Report Aggregation
```javascript
db.messages.aggregate([
  // 1. Filter first — uses indexes
  { $match: { orgId: "org_abc", role: "assistant",
      timestamp: { $gte: ISODate("2024-01-01"), $lt: ISODate("2024-02-01") } } },

  // 2. Group by agent, compute all metrics
  { $group: {
      _id: "$agentId",
      totalMessages:     { $sum: 1 },
      totalInputTokens:  { $sum: "$usage.inputTokens" },
      totalCost:         { $sum: "$usage.cost" },
      avgResponseMs:     { $avg: "$toolCall.durationMs" },
      conversationIds:   { $addToSet: "$conversationId" } // unique set
  }},

  // 3. Compute conversation count from set size
  { $addFields: { totalConversations: { $size: "$conversationIds" } } },

  // 4. Join agent names
  { $lookup: { from: "agents", localField: "_id", foreignField: "_id", as: "agentInfo",
      pipeline: [{ $project: { name: 1 } }] } },  // sub-pipeline — fetch only needed fields

  // 5. Flatten array from $lookup
  { $unwind: { path: "$agentInfo", preserveNullAndEmpty: true } },

  // 6. Shape output
  { $project: { _id: 0, agentId: "$_id", agentName: "$agentInfo.name",
      totalConversations: 1, totalMessages: 1,
      totalTokens: { $add: ["$totalInputTokens", "$totalOutputTokens"] },
      totalCost: { $round: ["$totalCost", 4] },
      avgResponseMs: { $round: ["$avgResponseMs", 0] } } },

  // 7. Sort
  { $sort: { totalCost: -1 } }
]);
```

### Key Operators
```javascript
// Grouping accumulators
$sum, $avg, $min, $max       // numeric aggregation
$push                        // collect all values into array
$addToSet                    // collect unique values
{ $sum: 1 }                  // count documents

// Shaping
$project                     // include/exclude/rename/compute
$addFields                   // add computed fields, keep existing

// Array
$unwind                      // flatten array — one doc per element
$size                        // array length
$filter, $map                // transform arrays

// Date
$dateToString                // format: "%Y-%m-%d"
$year, $month, $dayOfMonth   // extract date parts

// Faceted search (multiple groupings in one query)
{ $facet: {
    byStatus: [{ $group: { _id: "$status", count: { $sum: 1 } } }],
    byModel:  [{ $group: { _id: "$model",  count: { $sum: 1 } } }]
}}
```

### Performance Rules
```
1. $match FIRST — filters before other stages, uses indexes
2. $project early — shrink document size flowing through
3. Avoid $lookup on large collections — pre-aggregate instead
4. Use allowDiskUse: true if pipeline exceeds 100MB memory
5. Always explain() in dev: .aggregate([...]).explain("executionStats")
   IXSCAN = good, COLLSCAN = needs an index
```

### Interview Answer
> "MongoDB's aggregation pipeline is a series of stages: $match first to filter using indexes, $group to bucket and compute metrics with accumulators, $lookup to join other collections, $unwind to flatten joined arrays, $project to shape output. For a production monthly report I'd use the pre-aggregated agent_stats collection updated on every message — avoiding aggregating millions of documents. $lookup is expensive at scale, which is why MongoDB schemas pre-compute summaries rather than joining at query time."

---

---

## 3. Indexing Strategy

### Why Three Separate Indexes Don't Work
MongoDB uses ONE index per query. It picks the most selective, then scans the rest in memory.
```
{ orgId: 'org_abc', agentId: 'agent123', timestamp: { $gte: startDate } }
→ picks orgId index → 500k docs → scans all 500k in memory for agentId + timestamp → SLOW
```

### Compound Index — The Fix
```javascript
db.messages.createIndex({ orgId: 1, agentId: 1, timestamp: -1 });
// orgId → 500k → agentId → 200 → timestamp range → 20 — all at index level
```

### ESR Rule — Field Order Matters
**Equality → Sort → Range**
```javascript
// Query: find({ orgId, agentId, timestamp: { $gte } }).sort({ timestamp: -1 })
// ✅ Correct
db.messages.createIndex({ orgId: 1, agentId: 1, timestamp: -1 });
//                         E         E            S+R
// ❌ Wrong — range first blocks equality fields
db.messages.createIndex({ timestamp: -1, orgId: 1, agentId: 1 });
```

### 5 Index Types

**Compound** — multiple fields, ESR order
```javascript
db.messages.createIndex({ orgId: 1, agentId: 1, timestamp: -1 });
```

**Partial** — only indexes matching documents, smaller + faster writes
```javascript
db.agents.createIndex({ orgId: 1, name: 1 }, { partialFilterExpression: { status: 'active' } });
// Query MUST include status: 'active' to use this index
```

**Sparse** — excludes documents where field doesn't exist
```javascript
db.messages.createIndex({ 'toolCall.name': 1 }, { sparse: true });
// Only indexes messages that have toolCall — most messages don't
```

**TTL** — auto-deletes documents after time period
```javascript
db.conversations.createIndex({ lastMessageAt: 1 }, { expireAfterSeconds: 7776000 }); // 90 days
```

**Text** — full-text search (use Atlas Search / Elasticsearch for production)
```javascript
db.agents.createIndex({ name: 'text', systemPrompt: 'text' });
db.agents.find({ $text: { $search: 'customer support' } });
```

### Verify with explain()
```javascript
db.messages.find({ orgId, agentId, timestamp: { $gte: startDate } })
           .explain("executionStats");

// ✅ Good
"stage": "IXSCAN"
totalDocsExamined ≈ nReturned   // examined close to what was returned

// ❌ Bad
"stage": "COLLSCAN"             // no index used
totalDocsExamined >> nReturned  // examining far more than returned
```

### Covered Query — Zero Disk Reads
```javascript
// All query fields AND projection fields are in the index
db.messages.createIndex({ orgId: 1, agentId: 1, timestamp: -1 });
db.messages.find({ orgId, agentId }, { timestamp: 1, _id: 0 });
// Served entirely from index B-tree — no document reads = fastest possible
```

### Index Bloat — Indexes Slow Writes
```
Each insert/update touches every index on the collection.
Keep only indexes that serve actual query patterns.
Drop unused: db.messages.dropIndex("timestamp_1")
```

### Interview Answer
> "Separate indexes are inefficient — MongoDB picks one per query and scans the rest in memory. Compound index fix follows ESR rule: Equality fields first, Sort next, Range last — maximises narrowing before hitting the range condition. Verify with explain(): look for IXSCAN and totalDocsExamined close to nReturned. Partial indexes for filtered subsets, TTL for auto-expiry, sparse for optional fields. Too many indexes hurts write throughput — keep only what serves real query patterns."

---

---

## 4. MongoDB Transactions

### Yes — ACID Transactions Since v4.0
Requires replica set (Atlas always is one). Standalone MongoDB: no transactions.

### Full Pattern
```javascript
async function createAgentWithResources(orgId, agentConfig) {
  const session = await mongoose.startSession();

  try {
    session.startTransaction({
      readConcern: { level: 'snapshot' },  // consistent view of data
      writeConcern: { w: 'majority' }      // confirmed by majority of replica nodes
    });

    // ALL operations pass session — mandatory
    const agent = await Agent.create([{ orgId, ...agentConfig }], { session });

    const kb = await KnowledgeBase.create([{
      orgId, agentId: agent[0]._id, name: 'Default KB'
    }], { session });

    // Conditional deduct — throws if insufficient
    const billing = await OrgBilling.findOneAndUpdate(
      { orgId, credits: { $gte: 10 } },
      { $inc: { credits: -10 } },
      { session, new: true }
    );
    if (!billing) throw new AppError('Insufficient credits', 402, 'INSUFFICIENT_CREDITS');

    await session.commitTransaction();    // all three written atomically
    return { agent: agent[0], kb: kb[0] };

  } catch (err) {
    await session.abortTransaction();    // none of the three visible — clean rollback
    throw err;
  } finally {
    session.endSession();               // always clean up
  }
}
```

### Production Shorthand — withTransaction (handles retry automatically)
```javascript
const session = await mongoose.startSession();
const result = await session.withTransaction(async () => {
  // Mongoose auto-retries on transient errors (WriteConflict, network blip)
  // auto-aborts on permanent errors
  const agent = await Agent.create([...], { session });
  const kb    = await KnowledgeBase.create([...], { session });
  return { agent: agent[0], kb: kb[0] };
});
session.endSession();
```

### Limitations
```
Max 60 seconds — no external API calls or slow ops inside transactions
16MB op log limit — don't modify thousands of docs in one transaction
No DDL — can't create collections/indexes inside
Performance cost — uses locking, reduces concurrency
Single document ops are ALWAYS atomic without transactions — no transaction needed
```

### When to Use vs Not Use
```
✅ Multi-document, multi-collection atomicity
✅ Financial: deduct credits + create order + update inventory
❌ Single document update — already atomic
❌ High-throughput paths — lock overhead hurts concurrency
❌ Operations that take > a few seconds
```

### Interview Answer
> "MongoDB has ACID transactions since v4.0, requiring a replica set. Pattern: start session, startTransaction, pass session to every operation, commitTransaction on success, abortTransaction in catch, endSession in finally. Use Mongoose's withTransaction in production — it handles transient error retries. For create agent + create KB + deduct credits: all pass the same session, any failure rolls all three back atomically. Key limits: 60-second max, no external API calls inside. Single-document updates are already atomic without transactions."
