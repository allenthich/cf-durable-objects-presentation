<script setup lang="ts">
import { ref } from 'vue'

interface ChatRow {
  id: number
  sender: string
  content: string
}

const isHibernating = ref(true)
const lastEvent = ref('Idle — evicted from RAM while Cloudflare edge holds 2 open sockets')

const rows = ref<ChatRow[]>([
  { id: 1, sender: 'Alice', content: 'Ship it today?' },
  { id: 2, sender: 'Bob', content: 'Tests are green.' },
])

const sampleMessages: Record<string, string[]> = {
  Alice: ['Deploying v2 now', 'Latency looks great', 'All checks passed'],
  Bob: ['Watching metrics', 'Zero cold-start drops', 'Nice and clean!'],
}

const cursors = ref<Record<string, number>>({ Alice: 0, Bob: 0 })

function sendMessage(sender: 'Alice' | 'Bob') {
  const wasSleeping = isHibernating.value
  isHibernating.value = false

  const pool = sampleMessages[sender]
  const text = pool[cursors.value[sender] % pool.length]
  cursors.value[sender] += 1

  rows.value = [
    ...rows.value.slice(-3),
    { id: rows.value[rows.value.length - 1].id + 1, sender, content: text },
  ]

  lastEvent.value = wasSleeping
    ? `Woke from hibernation → SQLite INSERT → broadcast to 2 sockets`
    : `webSocketMessage() → SQLite INSERT → broadcast to 2 sockets`
}

function hibernateNow() {
  isHibernating.value = true
  lastEvent.value = 'Hibernating — 0 MB RAM billed; client WebSockets stay connected'
}
</script>

<template>
  <div class="hib-layout" role="region" aria-label="Interactive WebSocket Hibernation simulator">
    <div class="cf-panel clients-col">
      <div class="col-title">Connected Clients</div>

      <div v-for="user in (['Alice', 'Bob'] as const)" :key="user" class="socket-card">
        <div class="socket-header">
          <span class="user-label">{{ user }}</span>
          <span class="ws-open-badge">WS OPEN</span>
        </div>
        <div class="latest-feed">
          Latest: "{{ rows[rows.length - 1]?.content }}"
        </div>
        <button type="button" class="action-btn primary" @click="sendMessage(user)">
          Send as {{ user }}
        </button>
      </div>
    </div>

    <div class="cf-panel do-box" :class="{ sleeping: isHibernating }">
      <div class="do-header">
        <div>
          <div class="do-title">ChatRoom("general")</div>
          <div class="do-sub">{{ lastEvent }}</div>
        </div>

        <div class="status-controls">
          <span class="state-badge" :class="isHibernating ? 'asleep' : 'awake'">
            {{ isHibernating ? 'HIBERNATING · 0 MB RAM' : 'ACTIVE · IN MEMORY' }}
          </span>
          <button
            type="button"
            class="action-btn"
            :disabled="isHibernating"
            @click="hibernateNow"
          >
            Simulate Idle
          </button>
        </div>
      </div>

      <div class="sqlite-box">
        <div class="sqlite-header">
          <span>Embedded SQLite (<code>ctx.storage.sql</code>)</span>
          <span>Persists across hibernation</span>
        </div>

        <table class="sql-table">
          <thead>
            <tr>
              <th>id</th>
              <th>sender</th>
              <th>content</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="row in rows" :key="row.id">
              <td>{{ row.id }}</td>
              <td>{{ row.sender }}</td>
              <td>{{ row.content }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<style scoped>
.hib-layout {
  display: grid;
  grid-template-columns: 0.85fr 1.45fr;
  gap: 1rem;
  margin-top: 0.5rem;
}

.col-title {
  font-size: 0.75rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  margin-bottom: 0.625rem;
}

.clients-col {
  display: flex;
  flex-direction: column;
  gap: 0.625rem;
}

.socket-card {
  padding: 0.625rem;
  border-radius: 0.375rem;
  background: var(--cf-surface-elevated);
  border: 1px solid var(--cf-border);
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}

.socket-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.user-label {
  font-size: 0.85rem;
  font-weight: 700;
}

.ws-open-badge {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: 0.65rem;
  font-weight: 700;
  padding: 0.1rem 0.4rem;
  border-radius: 0.25rem;
  background: var(--cf-success-soft);
  color: var(--cf-success);
}

.latest-feed {
  font-size: 0.72rem;
  color: var(--cf-muted);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.do-box {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  gap: 0.75rem;
  border-color: var(--cf-accent-border);
  transition: border-color 0.2s ease, opacity 0.2s ease;
}

.do-box.sleeping {
  border-style: dashed;
  border-color: var(--cf-border);
}

.do-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 0.75rem;
}

.do-title {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: 0.92rem;
  font-weight: 700;
}

.do-sub {
  font-size: 0.72rem;
  color: var(--cf-muted);
  margin-top: 0.15rem;
}

.status-controls {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 0.375rem;
  flex-shrink: 0;
}

.state-badge {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: 0.68rem;
  font-weight: 700;
  padding: 0.15rem 0.5rem;
  border-radius: 0.25rem;
}

.state-badge.awake {
  background: var(--cf-accent-soft);
  border: 1px solid var(--cf-accent-border);
  color: var(--cf-accent);
}

.state-badge.asleep {
  background: var(--cf-surface-elevated);
  border: 1px solid var(--cf-border);
  color: var(--cf-muted);
}

.action-btn {
  padding: 0.3rem 0.625rem;
  font-size: 0.74rem;
  font-weight: 600;
  border-radius: 0.25rem;
  border: 1px solid var(--cf-border);
  background: var(--cf-surface);
  color: var(--cf-text);
  cursor: pointer;
}

.action-btn.primary {
  background: var(--cf-accent-soft);
  border-color: var(--cf-accent-border);
  color: var(--cf-accent);
}

.action-btn:hover:not(:disabled) {
  border-color: var(--cf-accent);
}

.action-btn:focus-visible {
  outline: 2px solid var(--cf-accent);
  outline-offset: 2px;
}

.action-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.sqlite-box {
  background: var(--cf-surface-elevated);
  border: 1px solid var(--cf-border);
  border-radius: 0.375rem;
  padding: 0.5rem 0.625rem;
}

.sqlite-header {
  display: flex;
  justify-content: space-between;
  font-size: 0.7rem;
  color: var(--cf-muted);
  margin-bottom: 0.375rem;
}

.sql-table {
  width: 100%;
  border-collapse: collapse;
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: 0.72rem;
}

.sql-table th,
.sql-table td {
  text-align: left;
  padding: 0.2rem 0.35rem;
  border-bottom: 1px solid var(--cf-border);
}

.sql-table th {
  color: var(--cf-muted);
  font-weight: 600;
}

.sql-table tr:last-child td {
  border-bottom: none;
}
</style>
