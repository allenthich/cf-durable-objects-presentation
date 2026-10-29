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
      <div class="text-xs opacity-70 mt-1">Global single-point coordination per room, doc, or session.</div>
    </div>
    <div class="cf-panel">
      <div class="text-xs font-mono uppercase tracking-wider opacity-60">Storage</div>
      <div class="font-semibold text-sm mt-1">Co-located SQLite</div>
      <div class="text-xs opacity-70 mt-1">Strongly consistent SQL living on the exact same thread.</div>
    </div>
    <div class="cf-panel">
      <div class="text-xs font-mono uppercase tracking-wider opacity-60">Real-Time</div>
      <div class="font-semibold text-sm mt-1">Hibernatable WebSockets</div>
      <div class="text-xs opacity-70 mt-1">Sleep at 0 MB RAM when idle while client sockets stay open.</div>
    </div>
  </div>

  <div class="flex items-center justify-between text-xs opacity-60 pt-4 border-t border-slate-500/20">
    <span>Reference: Cloudflare Developers — <em>Durable Objects Finally Make Sense</em></span>
    <span class="font-mono">Press Space / → to navigate</span>
  </div>
</div>

<!--
Welcome! Today we're demystifying Cloudflare Durable Objects in four practical parts:
1. Why stateless Workers + a database still leave a coordination gap.
2. Building a persistent chat room with RPC and embedded SQLite.
3. Upgrading it to real-time WebSockets with hibernation.
4. Architectural rules of thumb—how to partition and adopt selectively.
-->

---

<div class="cf-eyebrow">01 · The Core Problem</div>

## Why Stateless Workers + a Database Aren't Enough

<p class="cf-subtitle">
  Real-time and collaborative features need more than somewhere to save rows—they need a coordinator.
</p>

<div class="grid grid-cols-3 gap-4 mt-4">
  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="flex items-center gap-2 font-semibold text-base">
        <div class="i-lucide-zap text-amber-500" />
        <span>Stateless Workers</span>
      </div>
      <p class="text-sm opacity-80 mt-2">
        Execute close to every user, scale instantly, and <strong>forget in-memory state</strong> across requests.
      </p>
    </div>
    <div class="mt-4 pt-3 border-t border-slate-500/20 text-xs font-mono opacity-70">
      ✗ Cannot hold live room membership
    </div>
  </div>

  <div class="cf-panel flex flex-col justify-between">
    <div>
      <div class="flex items-center gap-2 font-semibold text-base">
        <div class="i-lucide-database text-sky-500" />
        <span>D1 / KV Databases</span>
      </div>
      <p class="text-sm opacity-80 mt-2">
        Answer <em>where</em> data is persisted, not <strong>who is in the room right now</strong> or how to push live updates.
      </p>
    </div>
    <div class="mt-4 pt-3 border-t border-slate-500/20 text-xs font-mono opacity-70">
      ✗ Requires external pub/sub to coordinate
    </div>
  </div>

  <div class="cf-panel-accent flex flex-col justify-between">
    <div>
      <div class="flex items-center gap-2 font-semibold text-base">
        <div class="i-lucide-box text-orange-500" />
        <span>Durable Object</span>
      </div>
      <p class="text-sm opacity-90 mt-2">
        A <strong>stateful serverless class</strong> combining single-threaded execution, in-memory state, and private storage.
      </p>
    </div>
    <div class="mt-4 pt-3 border-t border-orange-500/30 text-xs font-mono font-semibold text-orange-600 dark:text-orange-400">
      ✓ Compute + state + coordination in one place
    </div>
  </div>
</div>

<div class="cf-panel mt-5 flex items-center justify-between text-sm">
  <span><strong>Core Definition:</strong> Think of a Durable Object as an addressable micro-server instantiated on demand per ID.</span>
  <span class="cf-pill">Strong Consistency</span>
</div>

<!--
Regular Workers are stateless: two users in Tokyo and London hit two different isolates.
Storing data in KV or D1 tells you where data lives on disk, but real-time collaboration also needs to know WHO is connected right now and serialize concurrent updates through a single point of truth.
-->

---

<div class="cf-eyebrow">01 · Mental Model</div>

## The Golden Rule: `One ID = One Durable Object`

<p class="cf-subtitle">
  Everyone targeting <code>"general"</code> routes to the exact same instance; <code>"incidents"</code> gets an isolated instance.
</p>

<RoomRoutingDemo />

<!--
This is the single most important mental model: One ID equals One Durable Object instance globally.
Try clicking the room buttons on the left: when Alice in Tokyo and Bob in London both request #general, Cloudflare routes both to the exact same ChatRoom("general") instance. Meanwhile #incidents lives in its own isolated instance and SQLite database.
-->

---

<div class="cf-eyebrow">02 · Base Implementation (HTTP + SQLite)</div>

## Step 1: Configure Binding & SQLite in `wrangler.jsonc`

<p class="cf-subtitle">
  Declare your Durable Object class binding and enable embedded SQLite storage via a migration.
</p>

<div class="grid grid-cols-12 gap-5 items-start mt-2">
  <div class="col-span-5 flex flex-col gap-3">
    <div class="cf-panel">
      <div class="font-semibold text-sm">1. Worker Binding (`CHAT_ROOM`)</div>
      <p class="text-xs opacity-80 mt-1">
        Exposes <code>env.CHAT_ROOM</code> to your Worker so it can look up Durable Object stubs by name or ID.
      </p>
    </div>
    <div class="cf-panel">
      <div class="font-semibold text-sm">2. SQLite Migration (`new_sqlite_classes`)</div>
      <p class="text-xs opacity-80 mt-1">
        Provisions a private SQLite database attached directly to each <code>ChatRoom</code> instance—no network hop.
      </p>
    </div>
  </div>

  <div class="col-span-7">

```jsonc {5-12|13-18|all}
{
  "name": "durable-chat",
  "main": "src/index.ts",
  "compatibility_date": "2026-04-01",
  "durable_objects": {
    "bindings": [
      {
        "name": "CHAT_ROOM",
        "class_name": "ChatRoom"
      }
    ]
  },
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["ChatRoom"]
    }
  ]
}
```

  </div>
</div>

<!--
Setup in wrangler.jsonc takes two blocks:
1. Bind CHAT_ROOM to our exported ChatRoom class.
2. Add a migration tagging ChatRoom in new_sqlite_classes so every instance gets embedded SQLite storage.
-->

---

<div class="cf-eyebrow">02 · Base Implementation (HTTP + SQLite)</div>

## Step 2: Route by Room ID & Call RPC Methods

<p class="cf-subtitle">
  The stateless Worker extracts the room name, resolves the deterministic stub, and calls typed RPC methods.
</p>

```ts {1-6|8-10|12-20|all}
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const match = url.pathname.match(/^\/rooms\/([^/]+)\/messages$/);
    if (!match) return new Response("Not found", { status: 404 });
    const roomId = match[1];

    // Deterministic lookup: same roomId -> same global instance
    const id = env.CHAT_ROOM.idFromName(roomId);
    const stub = env.CHAT_ROOM.get(id);

    // Direct RPC calls to methods on the ChatRoom class
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
Notice how clean Worker-to-Durable-Object communication is with Workers RPC:
1. Parse roomId from the URL.
2. Call idFromName(roomId) + env.CHAT_ROOM.get(id) to get a typed stub.
3. Call stub.getMessages() or stub.sendMessage() like regular async TypeScript methods—no internal HTTP serialization boilerplate.
-->

---

<div class="cf-eyebrow">02 · Base Implementation (HTTP + SQLite)</div>

## Step 3: Persist State Inside the `ChatRoom` Class

<p class="cf-subtitle">
  Initialize the schema once in the constructor, then query SQLite synchronously inside RPC methods.
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
  <span><strong>Catch with HTTP-only:</strong> Persistence works, but connected users must <strong>manually refresh</strong> to see new messages.</span>
  <span class="cf-pill">Next: Add WebSockets</span>
</div>

<!--
Inside ChatRoom:
- Extend DurableObject from cloudflare:workers.
- Use blockConcurrencyWhile in the constructor to ensure the messages table exists before any RPC runs.
- Query and insert using ctx.storage.sql.exec().
The catch? Users still have to hit refresh in their browser to see messages from others. Let's fix that.
-->

---

<div class="cf-eyebrow">03 · Real-Time Coordination</div>

## Adding WebSockets—Without Redis or Extra Clusters

<p class="cf-subtitle">
  Because every user in <code>#general</code> already meets inside one Durable Object, that same object can coordinate sockets.
</p>

<div class="grid grid-cols-2 gap-4 mt-2">
  <div class="cf-panel">
    <div class="text-xs font-mono uppercase tracking-wider opacity-60">Typical Stack</div>
    <div class="font-semibold text-sm mt-1">4 Moving Parts to Sync</div>
    <ul class="text-xs space-y-1.5 mt-2 opacity-85">
      <li>Stateless API servers behind a load balancer</li>
      <li>Dedicated stateful WebSocket gateway fleet</li>
      <li>Redis Pub/Sub backplane to fan out room events</li>
      <li>External SQL database over TCP/connection pool</li>
    </ul>
  </div>

  <div class="cf-panel-accent">
    <div class="text-xs font-mono uppercase tracking-wider text-orange-600 dark:text-orange-400">Durable Object Stack</div>
    <div class="font-semibold text-sm mt-1">1 Self-Contained Instance per Room</div>
    <ul class="text-xs space-y-1.5 mt-2">
      <li><strong>Worker `/ws` route</strong> forwards the upgrade handshake</li>
      <li><strong>Same `ChatRoom` instance</strong> holds room WebSockets</li>
      <li><strong>Same `ChatRoom` instance</strong> writes to local SQLite</li>
      <li><strong>Hibernation API</strong> drops memory cost to zero when idle</li>
    </ul>
  </div>
</div>

<div class="mt-3">

```ts
// Add 4 lines in the Worker to forward WebSocket upgrades to the room's Durable Object:
if (url.pathname === `/rooms/${roomId}/ws` && request.headers.get("Upgrade") === "websocket") {
  return stub.fetch(request);
}
```

</div>

<!--
Normally, adding real-time chat means standing up WebSocket servers plus Redis Pub/Sub so Server A can tell Server B about a message.
With Durable Objects, everyone in #general is ALREADY routed to the same instance! We just add a /ws route in the Worker that forwards the HTTP Upgrade request directly to stub.fetch(request).
-->

---

<div class="cf-eyebrow">03 · High-Level Concept</div>

## How WebSocket Hibernation Works

<p class="cf-subtitle">
  The runtime host holds the open connections so your Durable Object can sleep for free whenever the room is quiet.
</p>

<HibernationConceptDiagram />

<!--
Here is the big idea in plain English—think of the runtime host like a receptionist holding the phone lines open:
1. Active Conversation: Messages pass straight through the runtime host to the Durable Object.
2. Room Goes Quiet: Instead of paying for an idle JavaScript instance to sit in memory holding open sockets, the Durable Object goes completely to sleep ($0 cost), while the runtime host keeps every user's WebSocket connected.
3. Someone Sends a Message: The moment anyone speaks, the runtime host wakes the Durable Object back up in milliseconds to save and broadcast—users never notice or reconnect.
-->

---

<div class="cf-eyebrow">03 · Hibernatable WebSocket API</div>

## Step 4: Accept, Hibernate, Wake & Broadcast

<p class="cf-subtitle">
  Press <kbd>Space</kbd> to see how <code>acceptWebSocket</code> and <code>webSocketMessage</code> complete real-time chat.
</p>

````md magic-move {lines: true}
```ts
export class ChatRoom extends DurableObject<Env> {
  // 1. Upgrade HTTP request & register a Hibernatable WebSocket
  async fetch(request: Request): Promise<Response> {
    const [client, server] = Object.values(new WebSocketPair());

    // Runtime host keeps the client socket open even if this DO hibernates!
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

  // 2. Automatically wakes on incoming frame -> persists -> broadcasts
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
First click: Instead of standard ws.accept(), we call this.ctx.acceptWebSocket(server). That opts into WebSocket Hibernation—when nobody is typing, the runtime evicts the Durable Object from memory so you stop paying for idle duration, while the runtime host keeps every client's WebSocket connection alive.
Second click: The moment anyone sends a frame, the runtime host wakes the object, invokes webSocketMessage(), saves to SQLite, and broadcasts to this.ctx.getWebSockets().
-->

---

<div class="cf-eyebrow">03 · Interactive Lifecycle</div>

## Live Walkthrough: Hibernation + Instant Broadcast

<p class="cf-subtitle">
  Try sending messages while hibernating, then click <strong>Simulate Idle</strong> to see sockets stay open at 0 MB RAM.
</p>

<HibernationDemo />

<!--
Here's what happens at runtime:
- Even when ChatRoom("general") is hibernating at 0 MB RAM, Alice and Bob's browser WebSockets remain OPEN on the runtime host.
- Click "Send as Alice": the Durable Object wakes in milliseconds, inserts the row into SQLite, and broadcasts to both clients.
- On the frontend, we replace HTTP polling with a standard WebSocket addEventListener("message") listener.
-->

---

<div class="cf-eyebrow">04 · Architectural Best Practices</div>

## Partition by Coordination Boundary—Never Centralize

<p class="cf-subtitle">
  Each Durable Object is intentionally single-threaded. Scale horizontally by creating millions of small instances.
</p>

<div class="grid grid-cols-2 gap-5 mt-3">
  <div class="cf-panel-accent">
    <div class="flex items-center justify-between">
      <span class="font-bold text-base text-orange-600 dark:text-orange-400">✓ DO: Granular Partitioning</span>
      <span class="cf-pill">Unbounded Horizontal Scale</span>
    </div>
    <p class="text-xs opacity-85 mt-2">
      Scope each Durable Object to the smallest unit that genuinely needs shared real-time coordination:
    </p>
    <ul class="text-xs space-y-1.5 mt-3">
      <li><code>ChatRoom("room_general")</code> — one per channel</li>
      <li><code>Document("doc_89a1")</code> — one per collaborative file</li>
      <li><code>GameMatch("match_402")</code> — one per active lobby</li>
      <li><code>AgentSession("user_19_run_4")</code> — one per stateful workflow</li>
    </ul>
  </div>

  <div class="cf-panel" style="border-color: rgba(239, 68, 68, 0.35);">
    <div class="flex items-center justify-between">
      <span class="font-bold text-base text-red-600 dark:text-red-400">✗ DON'T: Global Singleton</span>
      <span class="cf-pill">Throughput Bottleneck</span>
    </div>
    <p class="text-xs opacity-85 mt-2">
      Never funnel an entire multi-tenant application or global database through one Durable Object ID:
    </p>
    <ul class="text-xs space-y-1.5 mt-3 opacity-85">
      <li><code>AppDatabase("PROD_GLOBAL")</code> — serializes every user on Earth</li>
      <li><code>RateLimiter("ALL_USERS")</code> — turns edge traffic into a single queue</li>
      <li>Cross-room analytics queries — push aggregates to D1 / Analytics Engine instead</li>
    </ul>
  </div>
</div>

<!--
Because a single Durable Object is single-threaded to guarantee strong consistency without locks, you should never route your whole app through idFromName("global").
Instead, partition around natural coordination boundaries—one per chat room, document, match, or session. Ten thousand active rooms equals ten thousand parallel Durable Objects + ten thousand independent SQLite databases.
-->

---

<div class="cf-eyebrow">04 · Practical Adoption</div>

## Selective Adoption: Use Durable Objects Surgically

<p class="cf-subtitle">
  You don't need to migrate your entire backend to Cloudflare to benefit from Durable Objects.
</p>

<div class="grid grid-cols-12 gap-4 mt-3 items-stretch">
  <div class="col-span-6 cf-panel flex flex-col justify-between">
    <div>
      <div class="text-xs font-mono uppercase tracking-wider opacity-60">Keep Existing Core</div>
      <div class="font-semibold text-base mt-1">Your Existing App & Primary DB</div>
      <p class="text-xs opacity-80 mt-2">
        Keep auth, billing, relational reporting, and CRUD APIs in your current stack (Next.js, Rails, Go, Postgres, D1).
      </p>
    </div>
    <div class="pt-3 mt-4 border-t border-slate-500/20 text-xs font-mono opacity-75">
      Best for: Cross-tenant queries, joins, CRUD
    </div>
  </div>

  <div class="col-span-6 cf-panel-accent flex flex-col justify-between">
    <div>
      <div class="text-xs font-mono uppercase tracking-wider text-orange-600 dark:text-orange-400">Delegate Stateful Edge Paths</div>
      <div class="font-semibold text-base mt-1">Durable Objects Sidecar</div>
      <p class="text-xs opacity-90 mt-2">
        Point clients directly to a Worker + Durable Object solely for features requiring live presence or single-writer ordering:
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
A great practical takeaway from the video: Durable Objects aren't all-or-nothing.
You can keep your existing backend and database untouched, and introduce a small Worker + Durable Object service specifically for WebSockets, multiplayer presence, or per-session coordination.
-->

---

<div class="cf-eyebrow">Summary · Cheat Sheet</div>

## Durable Objects in Four Rules

<div class="grid grid-cols-2 gap-3.5 mt-4">
  <div class="cf-panel">
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">01 · ADDRESSING</div>
    <div class="font-semibold text-sm mt-1">One ID = One Global Instance</div>
    <p class="text-xs opacity-80 mt-1">
      <code>idFromName(roomId)</code> guarantees every caller for that room lands on the same single-threaded coordinator.
    </p>
  </div>

  <div class="cf-panel">
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">02 · PERSISTENCE</div>
    <div class="font-semibold text-sm mt-1">Private SQLite on the Same Thread</div>
    <p class="text-xs opacity-80 mt-1">
      <code>ctx.storage.sql.exec()</code> gives strongly consistent, zero-latency SQL storage right beside your code.
    </p>
  </div>

  <div class="cf-panel">
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">03 · REAL-TIME</div>
    <div class="font-semibold text-sm mt-1">Hibernatable WebSockets</div>
    <p class="text-xs opacity-80 mt-1">
      <code>ctx.acceptWebSocket()</code> keeps client sockets open in the runtime host while idle instances sleep for free.
    </p>
  </div>

  <div class="cf-panel">
    <div class="font-mono text-xs text-orange-600 dark:text-orange-400 font-bold">04 · ARCHITECTURE</div>
    <div class="font-semibold text-sm mt-1">Partition Granularly, Adopt Surgically</div>
    <p class="text-xs opacity-80 mt-1">
      Model one Durable Object per room/doc/session—never one global singleton—and adopt only where state is needed.
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
To wrap up:
1. One ID = One global instance.
2. Embedded SQLite gives zero-hop persistence.
3. Hibernatable WebSockets give real-time fan-out without paying for idle connections or running Redis.
4. Partition per room/document/session, and adopt selectively wherever coordination matters.
-->
