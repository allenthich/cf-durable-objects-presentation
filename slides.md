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

<div class="flex items-center gap-2 mb-2">
  <div class="cf-eyebrow !mb-0">
    <div class="i-lucide-message-square text-orange-500" />
    <span>Case Study · Real-Time Chat</span>
  </div>
  <span class="text-[11px] px-2.5 py-0.5 rounded-full bg-orange-500/10 text-orange-600 dark:text-orange-400 font-mono font-semibold border border-orange-500/20">
    Concrete Example
  </span>
</div>

## The Chat App Scaling Dilemma

<p class="cf-subtitle">
  How a simple team chat room gradually forces teams into complex infrastructure sprawl.
</p>

<div class="grid grid-cols-4 gap-3.5 mt-5 items-stretch">
  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">01 · POLLING</div>
      <div class="font-semibold text-sm mt-1">HTTP Polling</div>
      <p class="text-xs opacity-80 mt-2 leading-relaxed">
        Clients poll <code>GET /messages</code> every 3s.
      </p>
      <ul class="text-xs opacity-75 mt-2 space-y-1 pl-3.5 list-disc">
        <li>Constant read load on DB</li>
        <li>Delayed message delivery</li>
      </ul>
    </div>
    <div class="mt-4 pt-2.5 border-t border-slate-500/20 text-xs font-mono opacity-70">
      ✗ Stale UX &amp; DB load
    </div>
  </div>

  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">02 · WEBSOCKETS</div>
      <div class="font-semibold text-sm mt-1">Single Server</div>
      <p class="text-xs opacity-80 mt-2 leading-relaxed">
        In-memory socket array pushes events live.
      </p>
      <ul class="text-xs opacity-75 mt-2 space-y-1 pl-3.5 list-disc">
        <li>Instant local fanout</li>
        <li>Works great on <code>localhost</code></li>
      </ul>
    </div>
    <div class="mt-4 pt-2.5 border-t border-slate-500/20 text-xs font-mono opacity-70">
      ✗ Breaks past 1 server
    </div>
  </div>

  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">03 · MULTI-SERVER</div>
      <div class="font-semibold text-sm mt-1">Load Balancers</div>
      <p class="text-xs opacity-80 mt-2 leading-relaxed">
        Traffic is routed across multiple pods.
      </p>
      <ul class="text-xs opacity-75 mt-2 space-y-1 pl-3.5 list-disc">
        <li>Alice on Pod 1, Bob on Pod 2</li>
        <li>Pod 1 cannot reach Bob's socket</li>
      </ul>
    </div>
    <div class="mt-4 pt-2.5 border-t border-slate-500/20 text-xs font-mono opacity-70">
      ✗ Sockets isolated across pods
    </div>
  </div>

  <div class="cf-panel flex flex-col justify-between" style="border-color: rgba(239, 68, 68, 0.35);">
    <div>
      <div class="font-mono text-xs text-red-500 font-bold">04 · PUB/SUB FLEET</div>
      <div class="font-semibold text-sm mt-1 text-red-600 dark:text-red-400">Infra Sprawl</div>
      <p class="text-xs opacity-80 mt-2 leading-relaxed">
        Redis cluster synchronizes pods.
      </p>
      <ul class="text-xs opacity-75 mt-2 space-y-1 pl-3.5 list-disc">
        <li>Dedicated WebSocket fleet</li>
        <li>Paying 24/7 for idle socket RAM</li>
      </ul>
    </div>
    <div class="mt-4 pt-2.5 border-t border-red-500/25 text-xs font-mono opacity-80 text-red-600 dark:text-red-400">
      ✗ Accidental complexity
    </div>
  </div>
</div>

<!--
The classic chat app scaling journey:
1. HTTP Polling: Simplest start, but hammers the database and delivers delayed messages.
2. Single Server WebSockets: Fast and simple with an in-memory array, but only works on one machine.
3. Multi-Server: Once traffic scales past one pod, the load balancer splits Alice and Bob onto different servers. Neither can talk to the other.
4. Pub/Sub Sprawl: To fix it, teams deploy Redis brokers, WebSocket gateways, and pay 24/7 for idle connection RAM.

All you wanted was a chat room. You ended up managing a distributed pub/sub cluster.
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



<!--
Stateless Workers run across hundreds of PoPs without shared memory.
Databases persist state on disk, but cannot serialize concurrent edits or track active connections without extra infra like Redis.
A Durable Object fills this gap: one instance per entity with private storage and in-memory state.
-->


---

<div class="cf-eyebrow">01 · The Primitive</div>

## Anatomy of a Durable Object

<p class="cf-subtitle">
  Three core guarantees that make stateful coordination simple and safe.
</p>

<div class="grid grid-cols-3 gap-4 mt-6">
  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="flex items-center gap-2 font-semibold text-base">
        <div class="i-lucide-map-pin text-orange-500" />
        <span>Deterministic Addressing</span>
      </div>
      <p class="text-sm opacity-80 mt-2">
        Routed by unique ID or name (<code>idFromName("room-123")</code>). Every request worldwide for that ID routes to the <strong>exact same instance</strong>.
      </p>
    </div>
    <div class="mt-4 pt-3 border-t border-slate-500/20 text-xs font-mono text-orange-600 dark:text-orange-400">
      Single point of coordination
    </div>
  </div>

  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="flex items-center gap-2 font-semibold text-base">
        <div class="i-lucide-cpu text-blue-500" />
        <span>Single-Threaded Execution</span>
      </div>
      <p class="text-sm opacity-80 mt-2">
        Processes events <strong>one at a time</strong>. No race conditions, no distributed locks, no mutexes required across concurrent clients.
      </p>
    </div>
    <div class="mt-4 pt-3 border-t border-slate-500/20 text-xs font-mono text-blue-600 dark:text-blue-400">
      Zero concurrency races
    </div>
  </div>

  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="flex items-center gap-2 font-semibold text-base">
        <div class="i-lucide-database text-emerald-500" />
        <span>128 MB RAM + Private SQLite</span>
      </div>
      <p class="text-sm opacity-80 mt-2">
        Fast in-memory JS state (up to 128 MB heap ceiling) with disk-backed, zero-latency SQLite embedded right on the same thread.
      </p>
    </div>
    <div class="mt-4 pt-3 border-t border-slate-500/20 text-xs font-mono text-emerald-600 dark:text-emerald-400">
      In-memory speed + ACID durability
    </div>
  </div>
</div>

<!--
Three architectural pillars:
1. Deterministic Addressing: One ID = One instance worldwide. No routing ambiguity.
2. Single-Threaded: Strictly serial execution prevents race conditions.
3. 128 MB RAM + SQLite: Hot data lives in isolate memory, cold/durable data lives in embedded SQLite with zero network hops.
-->

---

<div class="cf-eyebrow">02 · Architecture</div>

## The Golden Rule: One ID = One Durable Object

<p class="cf-subtitle">
  Partition by domain entity, not by user. Each chat room, document, or game session gets its own isolated object.
</p>

<div class="mt-4">
  <RoomRoutingDemo />
</div>

<!--
The fundamental architectural decision:
- Durable Objects scale horizontally by having thousands or millions of small objects.
- Each object handles a single entity (e.g., room-alpha, room-beta).
- Requests for room-alpha always land on the room-alpha instance, wherever it lives globally.
-->

---

<div class="cf-eyebrow">02 · Implementation</div>

## Step 1: Route by Room ID & Call RPC Methods

<p class="cf-subtitle">
  Use <code>env.CHAT_ROOM.idFromName()</code> to locate the object, then call TypeScript RPC methods directly.
</p>

<div class="grid grid-cols-2 gap-4 mt-4">
  <div class="cf-panel">
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold mb-2">WORKER · ROUTING</div>
    <p class="text-sm opacity-80 mb-3">The Worker routes incoming HTTP requests to the target room's DO stub.</p>
    <div class="text-xs opacity-70">
      <strong>Key pattern:</strong> <code>idFromName(roomId)</code> produces a deterministic 64-character hex ID. The Worker never manages instance lifecycle.
    </div>
  </div>

  <div class="cf-panel">
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold mb-2">DURABLE OBJECT · EXPORT</div>
    <p class="text-sm opacity-80 mb-3">The class extends <code>DurableObject</code> and exposes standard async methods.</p>
    <div class="text-xs opacity-70">
      <strong>RPC calls:</strong> Methods like <code>getHistory()</code> are called directly over RPC with full type safety—no manual JSON fetch boilerplate.
    </div>
  </div>
</div>

```ts
// Worker entrypoint: route request to the target room
const id = env.CHAT_ROOM.idFromName(roomId)
const stub = env.CHAT_ROOM.get(id)

// Direct RPC call — type-safe, no manual fetch() serialization
const history = await stub.getHistory()
return Response.json(history)
```

<!--
Step 1: Routing & RPC.
Workers don't instantiate objects manually; they derive an ID from a name and obtain a stub.
Calls on the stub invoke methods on the remote Durable Object seamlessly via Cloudflare's Cap'n Proto RPC.
-->

---

<div class="cf-eyebrow">02 · Implementation</div>

## Step 2: Persist State Inside the ChatRoom Class

<p class="cf-subtitle">
  Every Durable Object has its own zero-latency, private SQLite database via <code>this.ctx.storage.sql</code>.
</p>

```ts
import { DurableObject } from 'cloudflare:workers'

export class ChatRoom extends DurableObject {
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env)
    // Run migrations on first startup — private SQLite database per room
    this.ctx.storage.sql.exec(`
      CREATE TABLE IF NOT EXISTS messages (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user TEXT NOT NULL,
        text TEXT NOT NULL,
        timestamp INTEGER NOT NULL
      )
    `)
  }

  async getHistory() {
    return this.ctx.storage.sql.exec('SELECT * FROM messages ORDER BY timestamp DESC LIMIT 50').toArray()
  }
}
```

<div class="grid grid-cols-2 gap-3 mt-2 text-xs">
  <div class="cf-panel">
    <strong>Synchronous SQLite:</strong> <code>sql.exec()</code> executes in-process on the same thread without network latency.
  </div>
  <div class="cf-panel">
    <strong>128 MB Heap Ceiling:</strong> In-memory JS variables have a 128 MB limit; large or durable histories live safely in SQLite.
  </div>
</div>

<!--
Step 2: Embedded SQLite persistence.
Unlike external SQL databases, this SQLite instance is private to this exact room.
It runs synchronously in the same thread with zero network latency.
-->

---

<div class="cf-eyebrow">02 · Architecture</div>

## Real-Time Architecture

<p class="cf-subtitle">
  Why Durable Objects collapse a 4-tier real-time stack into a single coordinated primitive.
</p>

<div class="grid grid-cols-2 gap-4 mt-3">
  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-slate-500 font-bold mb-2">TYPICAL ARCHITECTURE</div>
      <div class="space-y-1.5 text-xs opacity-80">
        <div class="p-2 rounded bg-slate-500/10 font-mono">1. Clients connect via WebSocket Gateway</div>
        <div class="p-2 rounded bg-slate-500/10 font-mono">2. Redis Pub/Sub brokers messages between nodes</div>
        <div class="p-2 rounded bg-slate-500/10 font-mono">3. Stateless Node.js / Go workers execute logic</div>
        <div class="p-2 rounded bg-slate-500/10 font-mono">4. External DB (PostgreSQL) stores durable data</div>
      </div>
      <div class="text-xs opacity-70 mt-3">
        <strong>Best for:</strong> Massive global broadcasts (100k+ viewers on 1 channel) and query-heavy analytics.
      </div>
    </div>
    <div class="mt-3 pt-2 border-t border-slate-500/20 text-xs font-mono opacity-70">
      ✗ 4 systems to monitor, scale, and pay for 24/7
    </div>
  </div>

  <div class="cf-panel-accent flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold mb-2">DURABLE OBJECT ARCHITECTURE</div>
      <div class="space-y-1.5 text-xs">
        <div class="p-2 rounded bg-orange-500/10 font-mono">1. Clients connect directly to room DO via WebSocket</div>
        <div class="p-2 rounded bg-orange-500/10 font-mono">2. Single-threaded &amp; synchronous: zero race conditions</div>
        <div class="p-2 rounded bg-orange-500/10 font-mono">3. Co-located in-memory state &amp; private SQLite</div>
        <div class="p-2 rounded bg-orange-500/10 font-mono">4. Hibernation: $0 compute cost when idle</div>
      </div>
      <div class="text-xs opacity-75 mt-3">
        <strong>Best for:</strong> Partitionable entities (chat rooms, documents, games, IoT devices, carts).
      </div>
    </div>
    <div class="mt-3 pt-2 border-t border-orange-500/30 text-xs font-mono font-semibold text-orange-600 dark:text-orange-400">
      ✓ One primitive replaces Gateway, Pub/Sub &amp; DB hops
    </div>
  </div>
</div>

<!--
Architecture comparison:
- Typical: Multi-tier sprawl (Gateways + Redis + Worker Fleets + DB). Great for 100k broadcast streams, but heavy for room-based apps.
- DO Architecture: Single-threaded and synchronous coordination right where connections live.
-->

---

<div class="cf-eyebrow">03 · Real-Time WebSockets</div>

## How WebSocket Hibernation Works

<p class="cf-subtitle">
  Keep thousands of connections open while paying only for active CPU execution time.
</p>

<div class="mt-2">
  <HibernationConceptDiagram />
</div>

<!--
WebSocket Hibernation solves the idle connection cost problem:
- Traditional servers charge for RAM and CPU as long as a socket stays open.
- Cloudflare offloads the TCP connection to the edge network infrastructure.
- The isolate shuts down and only wakes up when a frame arrives or an alarm fires.
-->

---

<div class="cf-eyebrow">03 · Real-Time WebSockets</div>

## Step 3: Accept, Hibernate, Wake & Broadcast

<p class="cf-subtitle">
  Use <code>ctx.acceptWebSocket()</code> to delegate socket management to the Cloudflare network.
</p>

```ts
export class ChatRoom extends DurableObject {
  async fetch(request: Request) {
    const pair = new WebSocketPair()
    // Accept socket into the DO hibernation system
    this.ctx.acceptWebSocket(pair[1])
    return new Response(null, { status: 101, webSocket: pair[0] })
  }

  async webSocketMessage(ws: WebSocket, message: string) {
    // DO automatically wakes from hibernation when a message arrives
    const data = JSON.parse(message)
    this.ctx.storage.sql.exec('INSERT INTO messages (user, text) VALUES (?, ?)', data.user, data.text)

    // Broadcast to all connected sockets in this room
    for (const client of this.ctx.getWebSockets()) {
      client.send(JSON.stringify(data))
    }
  }
}
```

<!--
Step 3: Hibernatable WebSockets in action.
- acceptWebSocket() registers the socket with the runtime.
- webSocketMessage() automatically wakes the DO only when an event occurs.
- getWebSockets() returns all active connections in this room.
-->

---

<div class="cf-eyebrow">03 · Real-Time WebSockets</div>

## Live Walkthrough: Hibernation + Instant Broadcast

<p class="cf-subtitle">
  Simulate real-time messaging, isolate hibernation sleep, and instant zero-cost wake.
</p>

<div class="mt-4">
  <HibernationDemo />
</div>

<!--
Interactive demo showing the lifecycle:
- Active: Processing messages, writing to SQLite.
- Idle: Sleeping, memory reclaimed, $0 compute charges.
- Wake: Sub-millisecond reactivation when a new message arrives.
-->

---

<div class="cf-eyebrow">04 · Scaling & Best Practices</div>

## Partitioning & Concurrency Model

<p class="cf-subtitle">
  Scale horizontally by partitioning across many objects. Avoid creating a single global bottleneck.
</p>

<div class="grid grid-cols-2 gap-4 mt-6">
  <div class="cf-panel-accent">
    <div class="font-mono text-xs text-emerald-600 dark:text-emerald-400 font-bold mb-2">DO THIS · GRANULAR PARTITIONING</div>
    <div class="text-sm font-semibold mb-1">One Object per Natural Domain Entity</div>
    <p class="text-xs opacity-80 leading-relaxed mb-3">
      Partition by chat room, document ID, shopping cart, or workspace. Each object handles its own traffic independently.
    </p>
    <div class="p-2 rounded bg-emerald-500/10 font-mono text-xs text-emerald-700 dark:text-emerald-300">
      env.CHAT_ROOM.idFromName(`room:${orgId}:${channelId}`)
    </div>
    <div class="mt-3 text-xs opacity-70">
      Scales horizontally to millions of concurrent rooms.
    </div>
  </div>

  <div class="cf-panel">
    <div class="font-mono text-xs text-red-500 font-bold mb-2">AVOID THIS · GLOBAL SINGLETON</div>
    <div class="text-sm font-semibold mb-1">One Giant Object for Everything</div>
    <p class="text-xs opacity-80 leading-relaxed mb-3">
      Funneling all app traffic through a single Durable Object creates a single-threaded CPU bottleneck and hits the 128 MB memory limit.
    </p>
    <div class="p-2 rounded bg-red-500/10 font-mono text-xs text-red-700 dark:text-red-300">
      env.CHAT_ROOM.idFromName("global-chat-master") // ✗ Anti-pattern
    </div>
    <div class="mt-3 text-xs opacity-70">
      Limits throughput to ~1,000 req/sec total.
    </div>
  </div>
</div>

<!--
Best practices for partitioning:
- The secret to DO scale is horizontal division of state.
- 10,000 rooms with 10 users each = 10,000 separate DOs running in parallel.
- Never use a DO as a global database replacement for your entire application.
-->

---

<div class="cf-eyebrow">04 · Scaling & Best Practices</div>

## Integration Strategy: Durable Objects Sidecar

<p class="cf-subtitle">
  You don't need to rewrite your entire backend. Keep your existing API and add DOs where coordination is needed.
</p>

<div class="grid grid-cols-3 gap-3.5 mt-6">
  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-slate-500 font-bold mb-2">EXISTING BACKEND</div>
      <div class="font-semibold text-sm mb-1">Auth &amp; Core Business Logic</div>
      <p class="text-xs opacity-80 leading-relaxed">
        Next.js, Rails, Django, Go API handling users, payments, and CRUD against PostgreSQL.
      </p>
    </div>
    <div class="mt-3 text-xs font-mono opacity-60">
      No migration needed
    </div>
  </div>

  <div class="cf-panel-accent flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold mb-2">DO COORDINATOR</div>
      <div class="font-semibold text-sm mb-1">Real-Time Sidecar</div>
      <p class="text-xs opacity-80 leading-relaxed">
        Durable Objects manage live WebSockets, presence, and transient state with zero infrastructure setup.
      </p>
    </div>
    <div class="mt-3 text-xs font-mono text-orange-600 dark:text-orange-400">
      Sidecar pattern
    </div>
  </div>

  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-slate-500 font-bold mb-2">DURABLE SYNC</div>
      <div class="font-semibold text-sm mb-1">Periodic Flush</div>
      <p class="text-xs opacity-80 leading-relaxed">
        DO flushes completed transcripts or summarized events to your primary Postgres DB via REST or webhooks.
      </p>
    </div>
    <div class="mt-3 text-xs font-mono opacity-60">
      Best of both worlds
    </div>
  </div>
</div>

<!--
Adoption strategy:
- You don't need a total rewrite.
- Use Durable Objects as a real-time sidecar alongside your existing Rails, Next.js, or Go backend.
- The DO manages the live connections and periodically flushes state to Postgres.
-->

---

<div class="cf-eyebrow">Summary</div>

## Durable Objects in Four Rules

<p class="cf-subtitle">
  Everything you need to remember about when and how to build with Durable Objects.
</p>

<div class="grid grid-cols-2 gap-4 mt-6">
  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">RULE 01</div>
      <div class="font-semibold text-sm mt-1">One ID = One Instance Globally</div>
      <p class="text-xs opacity-80 mt-1.5 leading-relaxed">
        <code>idFromName("room-123")</code> routes to the exact same isolate worldwide. Deterministic coordination without distributed locks.
      </p>
    </div>
  </div>

  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">RULE 02</div>
      <div class="font-semibold text-sm mt-1">Single-Threaded Execution</div>
      <p class="text-xs opacity-80 mt-1.5 leading-relaxed">
        Events run sequentially within each object. State changes and embedded SQLite queries execute safely with zero race conditions.
      </p>
    </div>
  </div>

  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">RULE 03</div>
      <div class="font-semibold text-sm mt-1">Hibernate to Eliminate Idle Costs</div>
      <p class="text-xs opacity-80 mt-1.5 leading-relaxed">
        Use <code>ctx.acceptWebSocket()</code> so Cloudflare holds TCP connections while the isolate sleeps. Pay $0 for idle connection RAM.
      </p>
    </div>
  </div>

  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">RULE 04</div>
      <div class="font-semibold text-sm mt-1">Partition by Entity, Not Globally</div>
      <p class="text-xs opacity-80 mt-1.5 leading-relaxed">
        Scale horizontally by scoping objects to natural entities (rooms, documents, carts). Never funnel your entire app into one singleton.
      </p>
    </div>
  </div>
</div>

<!--
Key takeaways:
1. One ID = One Object worldwide.
2. Single-threaded model removes race conditions.
3. Hibernation makes WebSockets cost-effective.
4. Scale horizontally by partitioning.
-->

---
layout: center
class: text-center
---

<div class="cf-eyebrow justify-center">Discussion &amp; Deep Dive</div>

# Questions & Next Steps

<p class="text-lg opacity-80 max-w-lg mx-auto mt-2">
  Explore the live demos or dive into the Cloudflare docs to start building stateful primitives.
</p>

<div class="grid grid-cols-3 gap-3.5 mt-8 max-w-2xl mx-auto text-left">
  <div class="cf-panel" v-click>
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">Q: WHAT ABOUT LATENCY?</div>
    <div class="font-semibold text-xs mt-1">Where Does the DO Run?</div>
    <p class="text-[11px] opacity-75 mt-1.5 leading-relaxed">
      Spins up near the first request or can be pinned via <code>locationHint</code> (e.g. <code>wnam</code>, <code>eeur</code>).
    </p>
  </div>

  <div class="cf-panel" v-click>
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">Q: MEMORY LIMITS?</div>
    <div class="font-semibold text-xs mt-1">128 MB Heap Ceiling</div>
    <p class="text-[11px] opacity-75 mt-1.5 leading-relaxed">
      In-memory JS state has a 128 MB limit. Large datasets should be stored in the private SQLite database.
    </p>
  </div>

  <div class="cf-panel" v-click>
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">Q: LARGE BROADCASTS?</div>
    <div class="font-semibold text-xs mt-1">1,000+ Sockets per Room</div>
    <p class="text-[11px] opacity-75 mt-1.5 leading-relaxed">
      For 100k+ spectator streams, fan out through Cloudflare Pub/Sub or Workers instead of 1 DO.
    </p>
  </div>
</div>

<div class="mt-8 flex justify-center gap-4 text-xs font-mono">
  <span class="opacity-60">developers.cloudflare.com/durable-objects</span>
</div>

<!--
Q&A wrap up:
- Address common architectural questions: geolocation/latency, memory boundaries, and broadcast limits.
-->
