<script setup lang="ts">
import { computed, ref } from 'vue'

type PhaseId = 'active' | 'hibernating' | 'waking'

interface PhaseInfo {
  id: PhaseId
  step: string
  label: string
  hostToDoLabel: string
  doStatus: string
  doDetail: string
  costBadge: string
  caption: string
}

const phases: PhaseInfo[] = [
  {
    id: 'active',
    step: '1',
    label: 'Active Conversation',
    hostToDoLabel: 'Forwarding messages',
    doStatus: 'AWAKE IN MEMORY',
    doDetail: 'Processes messages & saves to disk',
    costBadge: 'Billing: Active',
    caption: 'Active chat: messages flow directly through the runtime host to the Durable Object.',
  },
  {
    id: 'hibernating',
    step: '2',
    label: 'Room Goes Quiet (Hibernating)',
    hostToDoLabel: 'Paused — nothing to forward',
    doStatus: 'ASLEEP (UNLOADED)',
    doDetail: 'Evicted from RAM while room is silent',
    costBadge: 'Billing: $0',
    caption: 'Idle room: the host keeps all sockets open while the Durable Object unloads at $0 cost.',
  },
  {
    id: 'waking',
    step: '3',
    label: 'Someone Sends a Message',
    hostToDoLabel: 'Wakes & delivers message',
    doStatus: 'WAKES INSTANTLY',
    doDetail: 'Reloads state & broadcasts to room',
    costBadge: 'Billing: Active',
    caption: 'New message: the host wakes the Durable Object in milliseconds to persist and broadcast.',
  },
]

const current = ref<PhaseId>('hibernating')

const activePhase = computed(() => phases.find(p => p.id === current.value) ?? phases[1])
const isSleeping = computed(() => current.value === 'hibernating')
</script>

<template>
  <div class="concept-wrap" role="region" aria-label="WebSocket hibernation visual diagram">
    <!-- Phase selector -->
    <div class="phase-tabs" role="tablist" aria-label="Hibernation lifecycle stages">
      <button
        v-for="p in phases"
        :key="p.id"
        type="button"
        role="tab"
        class="phase-tab"
        :class="{ selected: current === p.id }"
        :aria-selected="current === p.id"
        @click="current = p.id"
      >
        <span class="step-num">{{ p.step }}</span>
        <span>{{ p.label }}</span>
      </button>
    </div>

    <!-- Main 3-node architecture diagram -->
    <div class="diagram-stage cf-panel">
      <div class="nodes-row">
        <!-- Node 1: Users -->
        <div class="node-card">
          <div class="node-eyebrow">Browsers</div>
          <div class="node-title">Users in Room</div>
          <div class="user-chips">
            <span class="user-chip"><span class="live-dot" /> Alice</span>
            <span class="user-chip"><span class="live-dot" /> Bob</span>
          </div>
          <div class="node-foot ok-text">Connected the whole time</div>
        </div>

        <!-- Pipe 1: Users <-> Runtime Host (Always Open!) -->
        <div class="pipe-col">
          <div class="pipe-pill solid-pill">Always Open</div>
          <svg viewBox="0 0 140 28" class="pipe-svg" aria-hidden="true">
            <line x1="6" y1="14" x2="134" y2="14" class="line-solid" />
            <polygon points="134,14 126,9 126,19" class="arrow-solid" />
            <polygon points="6,14 14,9 14,19" class="arrow-solid" />
          </svg>
          <div class="pipe-sub">WebSocket stays alive</div>
        </div>

        <!-- Node 2: Runtime Host -->
        <div class="node-card edge-card">
          <div class="node-eyebrow">Runtime Host</div>
          <div class="node-title">Connection Manager</div>
          <p class="node-desc">
            Holds open WebSockets in lightweight native memory while instances sleep.
          </p>
          <div class="node-foot">2 active sockets held</div>
        </div>

        <!-- Pipe 2: Runtime Host <-> Durable Object (Sleeps when idle!) -->
        <div class="pipe-col">
          <div class="pipe-pill" :class="isSleeping ? 'muted-pill' : 'accent-pill'">
            {{ isSleeping ? 'Idle · Disengaged' : 'Active Link' }}
          </div>
          <svg viewBox="0 0 140 28" class="pipe-svg" aria-hidden="true">
            <line
              x1="6"
              y1="14"
              x2="134"
              y2="14"
              :class="isSleeping ? 'line-dashed' : 'line-accent'"
            />
            <polygon
              v-if="!isSleeping"
              points="134,14 126,9 126,19"
              class="arrow-accent"
            />
          </svg>
          <div class="pipe-sub">{{ activePhase.hostToDoLabel }}</div>
        </div>

        <!-- Node 3: Durable Object -->
        <div class="node-card do-card" :class="{ asleep: isSleeping }">
          <div class="do-top">
            <span class="node-eyebrow">Durable Object</span>
            <span class="cost-tag" :class="isSleeping ? 'free' : 'paid'">
              {{ activePhase.costBadge }}
            </span>
          </div>
          <div class="node-title">ChatRoom</div>
          <div class="status-banner" :class="isSleeping ? 'banner-sleep' : 'banner-awake'">
            {{ activePhase.doStatus }}
          </div>
          <div class="node-foot">{{ activePhase.doDetail }}</div>
        </div>
      </div>

      <div class="caption-bar">
        {{ activePhase.caption }}
      </div>
    </div>
  </div>
</template>

<style scoped>
.concept-wrap {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  margin-top: 0.35rem;
}

.phase-tabs {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 0.5rem;
}

.phase-tab {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.45rem 0.75rem;
  border-radius: 0.375rem;
  border: 1px solid var(--cf-border);
  background: var(--cf-surface);
  color: var(--cf-muted);
  font-size: 0.78rem;
  font-weight: 600;
  cursor: pointer;
  text-align: left;
  transition: border-color 0.15s ease, background 0.15s ease, color 0.15s ease;
}

.phase-tab:hover {
  border-color: var(--cf-accent);
  color: var(--cf-text);
}

.phase-tab:focus-visible {
  outline: 2px solid var(--cf-accent);
  outline-offset: 2px;
}

.phase-tab.selected {
  background: var(--cf-accent-soft);
  border-color: var(--cf-accent);
  color: var(--cf-text);
}

.step-num {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 1.25rem;
  height: 1.25rem;
  border-radius: 9999px;
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: 0.7rem;
  font-weight: 700;
  background: var(--cf-surface-elevated);
  border: 1px solid var(--cf-border);
  flex-shrink: 0;
}

.phase-tab.selected .step-num {
  background: var(--cf-accent);
  border-color: var(--cf-accent);
  color: #ffffff;
}

.diagram-stage {
  padding: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.85rem;
}

.nodes-row {
  display: grid;
  grid-template-columns: 1fr 0.72fr 1.05fr 0.72fr 1.15fr;
  align-items: stretch;
  gap: 0.4rem;
}

.node-card {
  background: var(--cf-surface-elevated);
  border: 1px solid var(--cf-border);
  border-radius: 0.5rem;
  padding: 0.75rem;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  gap: 0.45rem;
  transition: opacity 0.2s ease, border-color 0.2s ease;
}

.edge-card {
  border-color: var(--cf-border);
}

.do-card {
  border-color: var(--cf-accent);
  background: var(--cf-accent-soft);
}

.do-card.asleep {
  border-style: dashed;
  border-color: var(--cf-border);
  background: var(--cf-surface);
  opacity: 0.8;
}

.do-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 0.25rem;
}

.node-eyebrow {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: 0.65rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: var(--cf-muted);
}

.node-title {
  font-size: 0.92rem;
  font-weight: 700;
}

.node-desc {
  font-size: 0.72rem;
  line-height: 1.4;
  color: var(--cf-muted);
  margin: 0 !important;
}

.user-chips {
  display: flex;
  gap: 0.375rem;
  flex-wrap: wrap;
}

.user-chip {
  display: inline-flex;
  align-items: center;
  gap: 0.3rem;
  padding: 0.15rem 0.45rem;
  border-radius: 0.25rem;
  font-size: 0.72rem;
  font-weight: 600;
  background: var(--cf-surface);
  border: 1px solid var(--cf-border);
}

.live-dot {
  width: 0.42rem;
  height: 0.42rem;
  border-radius: 9999px;
  background: var(--cf-success);
}

.status-banner {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: 0.7rem;
  font-weight: 700;
  padding: 0.25rem 0.45rem;
  border-radius: 0.25rem;
  text-align: center;
}

.banner-awake {
  background: var(--cf-surface-elevated);
  border: 1px solid var(--cf-accent-border);
  color: var(--cf-accent);
}

.banner-sleep {
  background: var(--cf-surface-elevated);
  border: 1px dashed var(--cf-border);
  color: var(--cf-muted);
}

.cost-tag {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: 0.63rem;
  font-weight: 700;
  padding: 0.1rem 0.35rem;
  border-radius: 0.25rem;
}

.cost-tag.free {
  background: var(--cf-success-soft);
  color: var(--cf-success);
}

.cost-tag.paid {
  background: var(--cf-surface-elevated);
  color: var(--cf-accent);
}

.node-foot {
  font-size: 0.68rem;
  color: var(--cf-muted);
  padding-top: 0.35rem;
  border-top: 1px solid var(--cf-border);
}

.ok-text {
  color: var(--cf-success);
  font-weight: 600;
}

.pipe-col {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
  gap: 0.2rem;
}

.pipe-svg {
  width: 100%;
  height: 1.6rem;
}

.line-solid {
  stroke: var(--cf-success);
  stroke-width: 2.5;
}

.arrow-solid {
  fill: var(--cf-success);
}

.line-accent {
  stroke: var(--cf-accent);
  stroke-width: 2.5;
}

.arrow-accent {
  fill: var(--cf-accent);
}

.line-dashed {
  stroke: var(--cf-muted);
  stroke-width: 2;
  stroke-dasharray: 5 5;
  opacity: 0.55;
}

.pipe-pill {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: 0.63rem;
  font-weight: 700;
  padding: 0.1rem 0.4rem;
  border-radius: 9999px;
  white-space: nowrap;
}

.solid-pill {
  background: var(--cf-success-soft);
  color: var(--cf-success);
}

.accent-pill {
  background: var(--cf-accent-soft);
  color: var(--cf-accent);
}

.muted-pill {
  background: var(--cf-surface-elevated);
  border: 1px solid var(--cf-border);
  color: var(--cf-muted);
}

.pipe-sub {
  font-size: 0.66rem;
  color: var(--cf-muted);
  line-height: 1.25;
}

.caption-bar {
  padding-top: 0.65rem;
  border-top: 1px solid var(--cf-border);
  font-size: 0.8rem;
  color: var(--cf-text);
  text-align: center;
}
</style>
