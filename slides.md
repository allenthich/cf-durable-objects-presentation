---
theme: default
title: Cloudflare Durable Objects — Clear, Simple, Practical
info: |
  A clear, practical walkthrough of Cloudflare Durable Objects:
  mental model, SQLite persistence, Hibernatable WebSockets, and architecture patterns.
favicon: /icon.svg
transition: slide-left
mdc: true
colorSchema: auto
---

<div class="h-full flex flex-col justify-between py-8">
  <div class="my-auto">
    <div class="cf-eyebrow">Cloudflare Primitives · Practical Guide</div>
    <h1 class="!text-6xl !leading-tight max-w-3xl mt-3">
      Cloudflare Durable Objects
    </h1>
    <p class="text-2xl opacity-75 max-w-2xl mt-4 leading-relaxed">
      Stateful serverless coordination, private SQLite, and real-time WebSockets—without extra infrastructure.
    </p>
  </div>

  <div class="text-xs opacity-60 pt-4 border-t border-slate-500/20">
    <span>Reference: Cloudflare Developers — <em>Durable Objects Finally Make Sense</em></span>
  </div>
</div>

<!--
Overview of Cloudflare Durable Objects in four parts:
1. Why stateless Workers + databases leave a coordination gap.
2. Building persistent rooms with RPC and embedded SQLite.
3. Adding real-time WebSockets with hibernation.
4. Partitioning patterns and selective adoption.
-->

---

<div class="cf-eyebrow">01 · The Core Problem</div>

## Why Stateless Workers + a Database Aren't Enough

<p class="cf-subtitle">
  Collaborative applications need more than persistent storage—they need a coordinator.
</p>

<div class="grid grid-cols-3 gap-4 mt-4">
  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="flex items-center gap-2 font-semibold text-base">
        <div class="i-lucide-zap text-amber-500" />
        <span>Stateless Workers</span>
      </div>
      <p class="text-sm opacity-80 mt-2">
        Execute close to users and scale instantly, but <strong>lose in-memory state</strong> between requests.
      </p>
    </div>
    <div class="mt-4 pt-3 border-t border-slate-500/20 text-xs font-mono opacity-70">
      ✗ No shared in-memory state
    </div>
  </div>

  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="flex items-center gap-2 font-semibold text-base">
        <div class="i-lucide-database text-sky-500" />
        <span>D1 / KV Databases</span>
      </div>
      <p class="text-sm opacity-80 mt-2">
        Persist data reliably, but cannot track <strong>who is currently connected</strong> or push live updates.
      </p>
    </div>
    <div class="mt-4 pt-3 border-t border-slate-500/20 text-xs font-mono opacity-70">
      ✗ Requires external pub/sub
    </div>
  </div>

  <div class="cf-panel-accent flex flex-col justify-between">
    <div>
      <div class="flex items-center gap-2 font-semibold text-base">
        <div class="i-lucide-box text-orange-500" />
        <span>Durable Object</span>
      </div>
      <p class="text-sm opacity-90 mt-2">
        A <strong>stateful serverless function</strong> combining single-threaded execution, in-memory state, and private storage.
      </p>
    </div>
    <div class="mt-4 pt-3 border-t border-orange-500/30 text-xs font-mono font-semibold text-orange-600 dark:text-orange-400">
      ✓ Compute + state + coordination
    </div>
  </div>
</div>

<div class="cf-panel mt-5 flex items-center justify-between text-sm">
  <span><strong>The Solution:</strong> A stateful serverless function combining compute, state, and coordination.</span>
  <span class="cf-pill">Next: Anatomy &amp; Constraints</span>
</div>

<!--
Stateless Workers run across hundreds of PoPs without shared memory.
Databases persist state on disk, but cannot serialize concurrent edits or track active connections without extra infra like Redis.
A Durable Object fills this gap: one instance per entity with private storage and in-memory state.
-->


---

<div class="cf-eyebrow">01 · The Primitive</div>

## Anatomy of a Durable Object

<p class="cf-subtitle">
  A stateful serverless function combining deterministic routing, single-threaded execution, and private storage.
</p>

<div class="grid grid-cols-3 gap-4 mt-5">
  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">01 · ADDRESSING</div>
      <div class="font-semibold text-base mt-1">Deterministic ID Routing</div>
      <p class="text-xs opacity-80 mt-2 leading-relaxed">
        Resolved globally by string name or unique ID (<code>idFromName("room_general")</code>). Every caller globally reaches the exact same coordinator.
      </p>
    </div>
    <div class="mt-4 pt-3 border-t border-slate-500/20 text-xs font-mono opacity-75">
      One ID = One Global Instance
    </div>
  </div>

  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">02 · EXECUTION</div>
      <div class="font-semibold text-base mt-1">Single-Threaded Execution</div>
      <p class="text-xs opacity-80 mt-2 leading-relaxed">
        Executes synchronously on one thread. Eliminates concurrency race conditions by design without locks (throughput constraint: ~1,000 req/s per entity).
      </p>
    </div>
    <div class="mt-4 pt-3 border-t border-slate-500/20 text-xs font-mono opacity-75">
      Lock-Free Strong Consistency
    </div>
  </div>

  <div class="cf-panel-accent flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">03 · STORAGE &amp; RAM</div>
      <div class="font-semibold text-base mt-1">128 MB RAM + Private SQLite</div>
      <p class="text-xs opacity-90 mt-2 leading-relaxed">
        Subject to a 128 MB V8 isolate heap ceiling. Use RAM as a lean cache, stream large payloads, and persist state to zero-latency, disk-backed SQLite.
      </p>
    </div>
    <div class="mt-4 pt-3 border-t border-orange-500/30 text-xs font-mono font-semibold text-orange-600 dark:text-orange-400">
      Zero-Hop Disk Persistence
    </div>
  </div>
</div>

<!--
The three core pillars of a Durable Object:
1. Deterministic Addressing: One ID = One global instance.
2. Single-Threaded Execution: Sequential execution guarantees strong consistency without distributed locks. (Constraint: ~1k req/s).
3. Memory & Storage: 128 MB heap ceiling is respected by treating RAM as an ephemeral cache and storing durable state in co-located SQLite.
-->

---

<div class="cf-eyebrow">01 · Mental Model</div>

## The Golden Rule: `One ID = One Durable Object`

<p class="cf-subtitle">
  Deterministic global routing: callers using the same ID reach the exact same coordinator.
</p>

<RoomRoutingDemo />

<!--
The fundamental mental model: One ID = One global instance.
Clicking the room buttons shows how Alice (Tokyo) and Bob (London) route to the exact same ChatRoom("general") instance, while #incidents runs in an independent instance with its own SQLite database.
-->

---

<div class="cf-eyebrow">02 · Base Implementation (HTTP + SQLite)</div>

## Step 1: Route by Room ID & Call RPC Methods

<p class="cf-subtitle">
  Extract the room name, resolve the deterministic stub, and call typed RPC methods.
</p>

```ts {1-6|8-10|12-20|all}
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const match = url.pathname.match(/^\/rooms\/([^/]+)\/messages$/);
    if (!match) return new Response("Not found", { status: 404 });
    const roomId = match[1];

    // Deterministic lookup: same roomId -> same instance
    const id = env.CHAT_ROOM.idFromName(roomId);
    const stub = env.CHAT_ROOM.get(id);

    // Call methods directly via Workers RPC
    if (request.method === "GET") {
      return Response.json(await stub.getMessages());
    }
    if (request.method === "POST") {
      const { sender, text } = await request.json<{ sender: string; text: string }>();
      return Response.json(await stub.sendMessage(sender, text));
    }
    return new Response("Method not allowed", { status: 405 });
  },
};
```

<!--
Workers RPC eliminates internal HTTP serialization boilerplate.
The Worker resolves the room ID, gets the stub, and calls stub.getMessages() or stub.sendMessage() as standard async TypeScript methods.
-->

---

<div class="cf-eyebrow">02 · Base Implementation (HTTP + SQLite)</div>

## Step 2: Persist State Inside the `ChatRoom` Class

<p class="cf-subtitle">
  Initialize schema on startup, then query SQLite synchronously inside RPC methods.
</p>

```ts {1-11|13-15|17-20|all}
import { DurableObject } from "cloudflare:workers";

export class ChatRoom extends DurableObject<Env> {
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    ctx.storage.sql.exec(`
      CREATE TABLE IF NOT EXISTS messages (
        id INTEGER PRIMARY KEY AUTOINCREMENT, sender TEXT, text TEXT
      )
    `);
  }

  async getMessages() {
    return this.ctx.storage.sql.exec("SELECT * FROM messages ORDER BY id ASC").toArray();
  }

  async sendMessage(sender: string, text: string) {
    this.ctx.storage.sql.exec("INSERT INTO messages (sender, text) VALUES (?, ?)", sender, text);
    return { ok: true };
  }
}
```

<div class="cf-panel mt-3 flex items-center justify-between text-xs">
  <span><strong>Limitation:</strong> Persistence works, but clients must <strong>poll or refresh</strong> for updates.</span>
  <span class="cf-pill">Next: WebSockets</span>
</div>

<!--
Inside ChatRoom:
- blockConcurrencyWhile ensures schema creation finishes before any method handles traffic.
- ctx.storage.sql.exec runs synchronously on the same thread without network hops.
- Next step: add WebSockets to push updates without polling.
-->

---

<div class="cf-eyebrow">03 · Real-Time Coordination</div>

## Real-Time Architecture

<p class="cf-subtitle">
  Eliminate the dedicated WebSocket fleet and Redis pub/sub backplane with a single coordination primitive.
</p>

<div class="grid grid-cols-2 gap-4 mt-2">
  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="text-xs font-mono uppercase tracking-wider opacity-60">Typical Architecture</div>
      <div class="font-semibold text-sm mt-1">4 Components to Synchronize</div>
      <ul class="text-xs space-y-1.5 mt-2 opacity-85">
        <li>Stateless API servers + load balancer</li>
        <li>Dedicated stateful WebSocket gateways</li>
        <li>Redis Pub/Sub backplane for event fanout</li>
        <li>External database with connection pooling</li>
      </ul>
    </div>
    <div class="mt-3 pt-2.5 border-t border-slate-500/20 text-xs opacity-75">
      <strong>Best for:</strong> Global broadcast (10k+ viewers), massive 1-to-many fanout, cross-table SQL joins.
    </div>
  </div>

  <div class="cf-panel-accent flex flex-col justify-between">
    <div>
      <div class="text-xs font-mono uppercase tracking-wider text-orange-600 dark:text-orange-400">Durable Object Architecture</div>
      <div class="font-semibold text-sm mt-1">1 Self-Contained Instance per Room</div>
      <ul class="text-xs space-y-1.5 mt-2">
        <li><strong>Colocated coordination:</strong> Same instance holds active room WebSockets</li>
        <li><strong>Lock-free consistency:</strong> Single-threaded execution eliminates race conditions</li>
        <li><strong>Private SQLite:</strong> Zero-latency SQL persistence on the exact same thread</li>
        <li><strong>Hibernation:</strong> Sockets stay open while idle instances sleep at $0</li>
        <li><strong>Zero Redis:</strong> No pub/sub fleet needed for in-room fanout</li>
      </ul>
    </div>
    <div class="mt-3 pt-2.5 border-t border-orange-500/30 text-xs text-orange-600 dark:text-orange-400 font-medium">
      <strong>Best for:</strong> Scoped collaboration (2–500 users), lock-free single-writer state, sparse idle sockets.
    </div>
  </div>
</div>



<!--
Architectural comparison:
- When to use Typical Architecture: Massive one-to-many fanout (10k+ listeners on one channel like a sports ticker) or complex cross-table SQL reporting.
- When to use Durable Objects: Scoped collaborative groups (chat channels, collaborative docs, game lobbies), single-writer state machines (auctions, seat reservations), and sparse connections where hibernation eliminates idle gateway costs.
- The 128 MB isolate limit is managed by persisting state directly to embedded SQLite rather than accumulating in-memory buffers.
-->

---

<div class="cf-eyebrow">03 · High-Level Concept</div>

## How WebSocket Hibernation Works

<p class="cf-subtitle">
  The runtime host keeps client sockets open while idle Durable Objects sleep at $0 cost.
</p>

<HibernationConceptDiagram />

<!--
Key concept:
1. Active Conversation: Messages flow directly through the host to the Durable Object.
2. Room Goes Quiet: Sockets remain open in lightweight native memory; the JS isolate is evicted ($0 compute cost).
3. Message Arrives: The host wakes the instance in milliseconds to persist and broadcast.
-->

---

<div class="cf-eyebrow">03 · Hibernatable WebSocket API</div>

## Step 3: Accept, Hibernate, Wake & Broadcast

<p class="cf-subtitle">
  Use <code>acceptWebSocket</code> to delegate sockets and <code>webSocketMessage</code> to handle incoming frames.
</p>

````md magic-move {lines: true}
```ts
export class ChatRoom extends DurableObject<Env> {
  // 1. Accept WebSocket & enable hibernation
  async fetch(request: Request): Promise<Response> {
    const [client, server] = Object.values(new WebSocketPair());

    // Host holds the connection open even if this instance hibernates
    this.ctx.acceptWebSocket(server);

    return new Response(null, { status: 101, webSocket: client });
  }
}
```

```ts
export class ChatRoom extends DurableObject<Env> {
  async fetch(request: Request): Promise<Response> {
    const [client, server] = Object.values(new WebSocketPair());
    this.ctx.acceptWebSocket(server);
    return new Response(null, { status: 101, webSocket: client });
  }

  // 2. Wakes on incoming frame -> persists -> broadcasts
  async webSocketMessage(ws: WebSocket, raw: string | ArrayBuffer) {
    const { sender, text } = JSON.parse(String(raw));
    this.ctx.storage.sql.exec("INSERT INTO messages (sender, text) VALUES (?, ?)", sender, text);

    const payload = JSON.stringify({ sender, text });
    for (const peer of this.ctx.getWebSockets()) {
      peer.send(payload);
    }
  }
}
```
````

<!--
First step: ctx.acceptWebSocket(server) registers the socket with the host runtime. When traffic stops, the object unloads from RAM without dropping clients.
Second step: When a message arrives, the object wakes, invokes webSocketMessage(), writes to SQLite, and broadcasts across this.ctx.getWebSockets().
-->

---

<div class="cf-eyebrow">03 · Interactive Lifecycle</div>

## Live Walkthrough: Hibernation + Instant Broadcast

<p class="cf-subtitle">
  Send messages while asleep to verify instant wake-up, or click <strong>Simulate Idle</strong> to hibernate.
</p>

<HibernationDemo />

<!--
Runtime behavior:
- During hibernation, sockets remain connected on the host at 0 MB RAM billed.
- Sending a message wakes the instance instantly, runs SQLite INSERT, and broadcasts.
- Clients experience seamless real-time chat with no reconnections.
-->

---

<div class="cf-eyebrow">04 · Architectural Best Practices</div>

## Partitioning & Concurrency Model

<p class="cf-subtitle">
  Single-threaded execution guarantees consistency. Scale horizontally by partitioning state across IDs.
</p>

<div class="grid grid-cols-2 gap-5 mt-3">
  <div class="cf-panel-accent">
    <div class="flex items-center justify-between">
      <span class="font-bold text-base text-orange-600 dark:text-orange-400">✓ DO: Granular Partitioning</span>
      <span class="cf-pill">Unbounded Scale</span>
    </div>
    <p class="text-xs opacity-85 mt-2">
      Scope instances to the smallest unit needing coordinated state:
    </p>
    <ul class="text-xs space-y-1.5 mt-3">
      <li><code>ChatRoom("room_general")</code> — one per channel</li>
      <li><code>Document("doc_89a1")</code> — one per document</li>
      <li><code>GameMatch("match_402")</code> — one per match lobby</li>
      <li><code>AgentSession("agent_run_4")</code> — one per workflow</li>
    </ul>
  </div>

  <div class="cf-panel" style="border-color: rgba(239, 68, 68, 0.35);">
    <div class="flex items-center justify-between">
      <span class="font-bold text-base text-red-600 dark:text-red-400">✗ DON'T: Global Singleton</span>
      <span class="cf-pill">Bottleneck</span>
    </div>
    <p class="text-xs opacity-85 mt-2">
      Never funnel global application traffic through a single ID:
    </p>
    <ul class="text-xs space-y-1.5 mt-3 opacity-85">
      <li><code>AppDatabase("GLOBAL")</code> — serializes all users in one queue</li>
      <li><code>RateLimiter("ALL_TRAFFIC")</code> — bottlenecks edge requests</li>
      <li>Cross-room queries — aggregate via D1 or Analytics Engine instead</li>
    </ul>
  </div>
</div>

<!--
Single-threaded execution guarantees lock-free consistency within an instance, eliminating concurrency races.
Each instance has a 128 MB memory limit: keep RAM lean, offload state to SQLite, and stream large payloads rather than buffering.
To scale, partition by entity: 10,000 active rooms run across 10,000 parallel Durable Objects.
-->

---

<div class="cf-eyebrow">04 · Practical Adoption</div>

## Integration Strategy

<p class="cf-subtitle">
  Run Durable Objects as a specialized coordination tier alongside your existing database and APIs.
</p>

<div class="grid grid-cols-12 gap-4 mt-3 items-stretch">
  <div class="col-span-6 cf-panel flex flex-col justify-between">
    <div>
      <div class="text-xs font-mono uppercase tracking-wider opacity-60">Keep Core Stack</div>
      <div class="font-semibold text-base mt-1">Existing App & Primary DB</div>
      <p class="text-xs opacity-80 mt-2">
        Keep auth, billing, CRUD APIs, and reporting in your current stack (Postgres, Next.js, Rails, Go).
      </p>
    </div>
    <div class="pt-3 mt-4 border-t border-slate-500/20 text-xs font-mono opacity-75">
      Best for: Relational queries, billing, CRUD
    </div>
  </div>

  <div class="col-span-6 cf-panel-accent flex flex-col justify-between">
    <div>
      <div class="text-xs font-mono uppercase tracking-wider text-orange-600 dark:text-orange-400">Delegate Stateful Edge Paths</div>
      <div class="font-semibold text-base mt-1">Durable Objects Sidecar</div>
      <p class="text-xs opacity-90 mt-2">
        Route real-time, presence, and lock-sensitive features directly to a Worker + Durable Object:
      </p>
      <div class="flex flex-wrap gap-1.5 mt-3">
        <span class="cf-pill">Live Chat & Presence</span>
        <span class="cf-pill">Multiplayer Cursors</span>
        <span class="cf-pill">AI Agent State</span>
        <span class="cf-pill">Per-Tenant Locks</span>
      </div>
    </div>
    <div class="pt-3 mt-4 border-t border-orange-500/30 text-xs font-mono font-semibold text-orange-600 dark:text-orange-400">
      Best for: WebSockets, coordination, zero-lock state
    </div>
  </div>
</div>

<!--
Durable Objects can be adopted incrementally as a specialized coordination tier alongside an existing Postgres/Rails/Next.js stack.
Use them surgically for real-time collaboration, cursors, presence, and stateful agents.
-->

---

<div class="cf-eyebrow">Summary · Cheat Sheet</div>

## Durable Objects in Four Rules

<div class="grid grid-cols-2 gap-4 mt-6">
  <div class="cf-panel py-4 flex flex-col justify-center">
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">01 · ADDRESSING</div>
    <div class="font-semibold text-base mt-1.5">One ID = One Global Instance</div>
  </div>

  <div class="cf-panel py-4 flex flex-col justify-center">
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">02 · PERSISTENCE</div>
    <div class="font-semibold text-base mt-1.5">Private SQLite on the Same Thread</div>
  </div>

  <div class="cf-panel py-4 flex flex-col justify-center">
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">03 · REAL-TIME</div>
    <div class="font-semibold text-base mt-1.5">Hibernatable WebSockets</div>
  </div>

  <div class="cf-panel py-4 flex flex-col justify-center">
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">04 · ARCHITECTURE</div>
    <div class="font-semibold text-base mt-1.5">Partition Granularly, Adopt Surgically</div>
  </div>
</div>

<div class="cf-panel-accent mt-4 flex items-center justify-between text-xs">
  <span>
    <strong>Watch the source video:</strong>
    <a href="https://www.youtube.com/watch?v=k4UXEfZf3sc" target="_blank" rel="noreferrer" class="underline ml-1">
      Cloudflare Developers — Durable Objects Finally Make Sense
    </a>
  </span>
  <span class="font-mono">Docs: developers.cloudflare.com/durable-objects</span>
</div>

<!--
Summary:
1. One ID = One global instance.
2. Embedded SQLite gives zero-hop persistence.
3. Hibernatable WebSockets provide cost-effective real-time fanout.
4. Partition per entity and adopt surgically alongside your primary stack.
-->

---

<div class="h-full flex flex-col justify-between py-4">
  <div>
    <div class="cf-eyebrow">Discussion &amp; Q&amp;A</div>
    <h1 class="!text-5xl !leading-tight mt-2">
      Questions?
    </h1>
  </div>

  <div v-click class="mt-4">
    <div class="text-xs font-mono uppercase tracking-wider text-orange-600 dark:text-orange-400 font-bold mb-3">
      Common Architectural Inquiries
    </div>
    <div class="grid grid-cols-2 gap-4">
      <div class="cf-panel">
        <div class="font-semibold text-sm text-orange-600 dark:text-orange-400">Where does an instance physically live?</div>
        <p class="text-xs opacity-80 mt-1">
          Spawned in the data center closest to the first request (or pinned to a specific jurisdiction/region). Callers route directly to it globally.
        </p>
      </div>
      <div class="cf-panel">
        <div class="font-semibold text-sm text-orange-600 dark:text-orange-400">How do we handle the 128 MB memory limit?</div>
        <p class="text-xs opacity-80 mt-1">
          Never buffer large payloads in RAM. Use <strong>streaming</strong> (<code>ReadableStream</code> / chunked WebSockets) so bytes pass through in small buffers, and persist history to embedded SQLite.
        </p>
      </div>
      <div class="cf-panel">
        <div class="font-semibold text-sm text-orange-600 dark:text-orange-400">What is the throughput ceiling &amp; concurrency model?</div>
        <p class="text-xs opacity-80 mt-1">
          Single-threaded execution guarantees lock-free consistency (~1,000 req/sec ceiling). Scale horizontally by partitioning IDs (per room, per doc).
        </p>
      </div>
      <div class="cf-panel">
        <div class="font-semibold text-sm text-orange-600 dark:text-orange-400">How do we run cross-room queries or analytics?</div>
        <p class="text-xs opacity-80 mt-1">
          DOs are isolated coordination shards. Checkpoint data or stream mutation events to D1, Postgres, or an analytical warehouse for reporting.
        </p>
      </div>
    </div>
  </div>

</div>

<!--
Start by pausing and inviting questions from the audience.
When ready, press Space or Right-Arrow to reveal the four most common architectural questions.
-->
