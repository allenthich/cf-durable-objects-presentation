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

<div class="h-full flex flex-col justify-between py-4">
  <div>
    <div class="cf-eyebrow">Cloudflare Primitives · Practical Guide</div>
    <h1 class="!text-5xl !leading-tight max-w-3xl mt-2">
      Cloudflare Durable Objects
    </h1>
    <p class="text-xl opacity-75 max-w-2xl mt-3">
      Stateful serverless coordination, private SQLite, and real-time WebSockets—without extra infrastructure.
    </p>
  </div>

  <div class="grid grid-cols-3 gap-4 mt-8">
    <div class="cf-panel">
      <div class="text-xs font-mono uppercase tracking-wider opacity-60">Mental Model</div>
      <div class="font-semibold text-sm mt-1">One ID = One Instance</div>
      <div class="text-xs opacity-70 mt-1">Single coordination point per room, document, or session.</div>
    </div>
    <div class="cf-panel">
      <div class="text-xs font-mono uppercase tracking-wider opacity-60">Storage</div>
      <div class="font-semibold text-sm mt-1">Co-located SQLite</div>
      <div class="text-xs opacity-70 mt-1">Zero-latency, strongly consistent SQL on the same thread.</div>
    </div>
    <div class="cf-panel">
      <div class="text-xs font-mono uppercase tracking-wider opacity-60">Real-Time</div>
      <div class="font-semibold text-sm mt-1">Hibernatable WebSockets</div>
      <div class="text-xs opacity-70 mt-1">Idle instances sleep at 0 MB RAM while sockets stay open.</div>
    </div>
  </div>

  <div class="flex items-center justify-between text-xs opacity-60 pt-4 border-t border-slate-500/20">
    <span>Reference: Cloudflare Developers — <em>Durable Objects Finally Make Sense</em></span>
    <span class="font-mono">Press Space / → to navigate</span>
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
  <span><strong>Core Definition:</strong> An addressable micro-server instantiated on demand per unique ID.</span>
  <span class="cf-pill">Strong Consistency</span>
</div>

<!--
Stateless Workers run across hundreds of PoPs without shared memory.
Databases persist state on disk, but cannot serialize concurrent edits or track active connections without extra infra like Redis.
A Durable Object fills this gap: one instance per entity with private storage and in-memory state.
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

```ts {1-12|14-24|all}
import { DurableObject } from "cloudflare:workers";

export class ChatRoom extends DurableObject<Env> {
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    ctx.blockConcurrencyWhile(async () => {
      ctx.storage.sql.exec(`
        CREATE TABLE IF NOT EXISTS messages (
          id INTEGER PRIMARY KEY AUTOINCREMENT, sender TEXT, text TEXT
        )
      `);
    });
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
  <div class="cf-panel">
    <div class="text-xs font-mono uppercase tracking-wider opacity-60">Typical Architecture</div>
    <div class="font-semibold text-sm mt-1">4 Components to Synchronize</div>
    <ul class="text-xs space-y-1.5 mt-2 opacity-85">
      <li>Stateless API servers + load balancer</li>
      <li>Dedicated stateful WebSocket gateways</li>
      <li>Redis Pub/Sub backplane for event fanout</li>
      <li>External database with connection pooling</li>
    </ul>
  </div>

  <div class="cf-panel-accent">
    <div class="text-xs font-mono uppercase tracking-wider text-orange-600 dark:text-orange-400">Durable Object Architecture</div>
    <div class="font-semibold text-sm mt-1">1 Self-Contained Instance per Room</div>
    <ul class="text-xs space-y-1.5 mt-2">
      <li><strong>Worker `/ws` route</strong> forwards the upgrade handshake</li>
      <li><strong>Same instance</strong> holds room WebSockets</li>
      <li><strong>Same instance</strong> persists to local SQLite</li>
      <li><strong>Hibernation</strong> drops idle memory cost to $0</li>
    </ul>
  </div>
</div>

<div class="mt-3">

```ts
// Forward WebSocket upgrades directly to the room's Durable Object:
if (url.pathname === `/rooms/${roomId}/ws` && request.headers.get("Upgrade") === "websocket") {
  return stub.fetch(request);
}
```

</div>

<!--
Traditional real-time stacks require separate WebSocket fleets and Redis Pub/Sub to sync servers.
With Durable Objects, room members already route to the same coordinator, eliminating the pub/sub tier entirely.
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
  Single-threaded execution guarantees consistency within a 128 MB memory ceiling. Scale horizontally across IDs.
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

<div class="grid grid-cols-2 gap-3.5 mt-4">
  <div class="cf-panel">
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">01 · ADDRESSING</div>
    <div class="font-semibold text-sm mt-1">One ID = One Global Instance</div>
    <p class="text-xs opacity-80 mt-1">
      <code>idFromName(id)</code> routes all callers to the same single-threaded coordinator.
    </p>
  </div>

  <div class="cf-panel">
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">02 · PERSISTENCE</div>
    <div class="font-semibold text-sm mt-1">Private SQLite on the Same Thread</div>
    <p class="text-xs opacity-80 mt-1">
      <code>ctx.storage.sql.exec()</code> runs strongly consistent SQL on the exact same thread.
    </p>
  </div>

  <div class="cf-panel">
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">03 · REAL-TIME</div>
    <div class="font-semibold text-sm mt-1">Hibernatable WebSockets</div>
    <p class="text-xs opacity-80 mt-1">
      <code>ctx.acceptWebSocket()</code> keeps client sockets open while idle instances sleep for free.
    </p>
  </div>

  <div class="cf-panel">
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">04 · ARCHITECTURE</div>
    <div class="font-semibold text-sm mt-1">Partition Granularly, Adopt Surgically</div>
    <p class="text-xs opacity-80 mt-1">
      Partition per room, document, or session. Never funnel global traffic through one ID.
    </p>
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

  <div class="flex items-center justify-between text-xs opacity-60 pt-4 border-t border-slate-500/20">
    <span>Durable Objects: Stateful coordination at the edge</span>
    <span class="font-mono">Press Space / → to reveal common FAQs</span>
  </div>
</div>

<!--
Start by pausing and inviting questions from the audience.
When ready, press Space or Right-Arrow to reveal the four most common architectural questions.
-->
