<script setup lang="ts">
import { computed, ref } from 'vue'

type RoomId = 'general' | 'incidents'

interface EdgeClient {
  id: string
  name: string
  colo: string
  room: RoomId
}

const rooms: RoomId[] = ['general', 'incidents']

const clients = ref<EdgeClient[]>([
  { id: 'u1', name: 'Alice', colo: 'NRT · Tokyo', room: 'general' },
  { id: 'u2', name: 'Bob', colo: 'LHR · London', room: 'general' },
  { id: 'u3', name: 'Mei', colo: 'SFO · San Francisco', room: 'incidents' },
])

function setRoom(clientId: string, room: RoomId) {
  const target = clients.value.find(c => c.id === clientId)
  if (target) target.room = room
}

const grouped = computed(() => ({
  general: clients.value.filter(c => c.room === 'general'),
  incidents: clients.value.filter(c => c.room === 'incidents'),
}))
</script>

<template>
  <div class="routing-grid" role="region" aria-label="Interactive Durable Object routing simulator">
    <div class="cf-panel">
      <div class="panel-header">
        <span class="panel-label">1. Client Requests at Edge</span>
        <span class="panel-hint">Click a room to re-route</span>
      </div>

      <div class="client-list">
        <div v-for="client in clients" :key="client.id" class="client-row">
          <div class="client-meta">
            <span class="client-name">{{ client.name }}</span>
            <span class="client-colo">{{ client.colo }}</span>
          </div>

          <div class="room-switcher" role="group" :aria-label="`Select room for ${client.name}`">
            <button
              v-for="r in rooms"
              :key="r"
              type="button"
              class="room-btn"
              :class="{ active: client.room === r }"
              :aria-pressed="client.room === r"
              @click="setRoom(client.id, r)"
            >
              #{{ r }}
            </button>
          </div>
        </div>
      </div>

      <div class="code-callout">
        <code>env.CHAT_ROOM.idFromName(roomId)</code>
        <span>Same name &rarr; same instance</span>
      </div>
    </div>

    <div class="instances-col">
      <div
        v-for="r in rooms"
        :key="r"
        class="cf-panel instance-card"
        :class="{ populated: grouped[r].length > 0 }"
      >
        <div class="instance-top">
          <div>
            <span class="instance-badge">Durable Object Instance</span>
            <div class="instance-title">ChatRoom("{{ r }}")</div>
          </div>
          <span class="count-pill">
            {{ grouped[r].length }} {{ grouped[r].length === 1 ? 'peer' : 'peers' }}
          </span>
        </div>

        <div class="peer-row">
          <template v-if="grouped[r].length > 0">
            <span v-for="peer in grouped[r]" :key="peer.id" class="peer-chip">
              <span class="dot" aria-hidden="true" />
              {{ peer.name }} ({{ peer.colo.split(' · ')[0] }})
            </span>
          </template>
          <span v-else class="empty-note">Idle — 0 active callers</span>
        </div>

        <div class="storage-bar">
          <span>Private SQLite: <code>{{ r }}.sqlite</code></span>
          <span>Single-threaded</span>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.routing-grid {
  display: grid;
  grid-template-columns: 1fr 1.1fr;
  gap: 1rem;
  margin-top: 0.5rem;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 0.75rem;
}

.panel-label {
  font-size: 0.78rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.panel-hint {
  font-size: 0.72rem;
  color: var(--cf-muted);
}

.client-list {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.client-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.5rem 0.625rem;
  background: var(--cf-surface-elevated);
  border: 1px solid var(--cf-border);
  border-radius: 0.375rem;
}

.client-meta {
  display: flex;
  flex-direction: column;
}

.client-name {
  font-size: 0.85rem;
  font-weight: 600;
}

.client-colo {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: 0.68rem;
  color: var(--cf-muted);
}

.room-switcher {
  display: flex;
  gap: 0.375rem;
}

.room-btn {
  padding: 0.25rem 0.5rem;
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: 0.72rem;
  font-weight: 600;
  border-radius: 0.25rem;
  border: 1px solid var(--cf-border);
  background: var(--cf-surface);
  color: var(--cf-muted);
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;
}

.room-btn:hover {
  border-color: var(--cf-accent);
  color: var(--cf-text);
}

.room-btn:focus-visible {
  outline: 2px solid var(--cf-accent);
  outline-offset: 2px;
}

.room-btn.active {
  background: var(--cf-accent-soft);
  border-color: var(--cf-accent);
  color: var(--cf-accent);
}

.code-callout {
  margin-top: 0.75rem;
  padding-top: 0.625rem;
  border-top: 1px solid var(--cf-border);
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.72rem;
  color: var(--cf-muted);
}

.code-callout code {
  font-size: 0.72rem;
  color: var(--cf-accent);
  font-weight: 600;
}

.instances-col {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.instance-card {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  transition: border-color 0.15s ease;
}

.instance-card.populated {
  border-color: var(--cf-accent-border);
}

.instance-top {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

.instance-badge {
  display: block;
  font-size: 0.65rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: var(--cf-accent);
}

.instance-title {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: 0.9rem;
  font-weight: 700;
}

.count-pill {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: 0.7rem;
  font-weight: 600;
  padding: 0.125rem 0.5rem;
  border-radius: 0.25rem;
  background: var(--cf-surface-elevated);
  border: 1px solid var(--cf-border);
}

.peer-row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.375rem;
  min-height: 1.625rem;
  align-items: center;
}

.peer-chip {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  font-size: 0.75rem;
  font-weight: 500;
  padding: 0.15rem 0.5rem;
  border-radius: 0.25rem;
  background: var(--cf-accent-soft);
  border: 1px solid var(--cf-accent-border);
}

.dot {
  width: 0.4rem;
  height: 0.4rem;
  border-radius: 9999px;
  background: var(--cf-accent);
}

.empty-note {
  font-size: 0.75rem;
  color: var(--cf-muted);
  font-style: italic;
}

.storage-bar {
  display: flex;
  justify-content: space-between;
  padding-top: 0.45rem;
  border-top: 1px solid var(--cf-border);
  font-size: 0.7rem;
  color: var(--cf-muted);
}
</style>
