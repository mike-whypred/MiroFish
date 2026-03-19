<template>
  <div class="leaderboard-view">
    <!-- Header -->
    <header class="app-header">
      <div class="header-left">
        <div class="brand" @click="router.push('/')">MIROFISH</div>
      </div>

      <div class="header-center">
        <h1 class="page-title">Agent Leaderboard</h1>
      </div>

      <div class="header-right">
        <button class="refresh-btn" @click="refreshData" :disabled="loading">
          <span v-if="loading">Loading...</span>
          <span v-else>Refresh</span>
        </button>
      </div>
    </header>

    <!-- Stats Overview -->
    <section class="stats-section">
      <div class="stat-card">
        <div class="stat-value">{{ stats.total_agents }}</div>
        <div class="stat-label">Total Agents</div>
      </div>
      <div class="stat-card">
        <div class="stat-value">{{ stats.scored_simulations }}</div>
        <div class="stat-label">Scored Simulations</div>
      </div>
      <div class="stat-card">
        <div class="stat-value">{{ stats.pending_simulations }}</div>
        <div class="stat-label">Pending</div>
      </div>
      <div class="stat-card highlight">
        <div class="stat-value">{{ (stats.avg_hit_rate * 100).toFixed(1) }}%</div>
        <div class="stat-label">Avg Hit Rate</div>
      </div>
      <div class="stat-card" v-if="stats.top_agent">
        <div class="stat-value top-agent">{{ stats.top_agent?.agent_id?.slice(0, 12) || 'N/A' }}</div>
        <div class="stat-label">Top Agent (Alpha: {{ (stats.top_agent?.alpha_score * 100).toFixed(1) }}%)</div>
      </div>
    </section>

    <!-- Main Content -->
    <main class="main-content">
      <!-- Left: Leaderboard Table -->
      <section class="leaderboard-section">
        <div class="section-header">
          <h2 class="section-title">Top Performing Agents</h2>
          <div class="filter-controls">
            <select v-model="topicFilter" @change="loadTopAgents" class="topic-select">
              <option value="">All Topics</option>
              <option v-for="topic in availableTopics" :key="topic" :value="topic">
                {{ topic }}
              </option>
            </select>
            <input
              type="number"
              v-model.number="agentLimit"
              @change="loadTopAgents"
              class="limit-input"
              min="5"
              max="100"
              placeholder="Limit"
            />
          </div>
        </div>

        <div class="table-container">
          <table class="leaderboard-table">
            <thead>
              <tr>
                <th class="rank-col">Rank</th>
                <th class="agent-col">Agent ID</th>
                <th class="persona-col">Persona</th>
                <th class="hit-rate-col">Hit Rate</th>
                <th class="alpha-col">Alpha</th>
                <th class="calibration-col">Calibration</th>
                <th class="sims-col">Sims</th>
                <th class="topics-col">Best Topics</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="agent in topAgents"
                :key="agent.agent_id"
                @click="showAgentDetail(agent.agent_id)"
                class="agent-row"
              >
                <td class="rank-col">
                  <span class="rank-badge" :class="getRankClass(agent.rank)">
                    {{ agent.rank }}
                  </span>
                </td>
                <td class="agent-col">
                  <span class="agent-id">{{ agent.agent_id.slice(0, 16) }}</span>
                </td>
                <td class="persona-col">
                  <span class="persona-text">{{ truncatePersona(agent.persona) }}</span>
                </td>
                <td class="hit-rate-col">
                  <div class="metric-bar">
                    <div class="bar-fill hit-rate" :style="{ width: (agent.hit_rate * 100) + '%' }"></div>
                    <span class="metric-value">{{ (agent.hit_rate * 100).toFixed(1) }}%</span>
                  </div>
                </td>
                <td class="alpha-col">
                  <span class="alpha-value" :class="{ positive: agent.alpha_score > 0, negative: agent.alpha_score < 0 }">
                    {{ agent.alpha_score > 0 ? '+' : '' }}{{ (agent.alpha_score * 100).toFixed(1) }}%
                  </span>
                </td>
                <td class="calibration-col">
                  <span class="calibration-value">{{ agent.calibration_score.toFixed(3) }}</span>
                </td>
                <td class="sims-col">
                  <span class="sims-value">{{ agent.total_simulations }}</span>
                </td>
                <td class="topics-col">
                  <div class="topic-tags">
                    <span
                      v-for="topic in agent.best_topics.slice(0, 2)"
                      :key="topic"
                      class="topic-tag"
                    >
                      {{ topic }}
                    </span>
                  </div>
                </td>
              </tr>
            </tbody>
          </table>

          <div v-if="topAgents.length === 0 && !loading" class="empty-state">
            <p>No agents found. Run simulations to populate the leaderboard.</p>
          </div>
        </div>
      </section>

      <!-- Right: Simulations List -->
      <section class="simulations-section">
        <div class="section-header">
          <h2 class="section-title">Simulation History</h2>
          <div class="filter-controls">
            <select v-model="simStatusFilter" @change="loadSimulations" class="status-select">
              <option value="">All Status</option>
              <option value="pending">Pending</option>
              <option value="scored">Scored</option>
            </select>
          </div>
        </div>

        <div class="simulations-list">
          <div
            v-for="sim in simulations"
            :key="sim.simulation_id"
            class="simulation-card"
          >
            <div class="sim-header">
              <span class="sim-id">{{ sim.simulation_id.slice(0, 12) }}</span>
              <span class="sim-status" :class="sim.scored ? 'scored' : 'pending'">
                {{ sim.scored ? 'Scored' : 'Pending' }}
              </span>
            </div>
            <p class="sim-question">{{ truncateQuestion(sim.question) }}</p>
            <div class="sim-meta">
              <span class="agent-count">{{ sim.agent_count }} agents</span>
              <span v-if="sim.topic_tags?.length" class="sim-topics">
                {{ sim.topic_tags.slice(0, 2).join(', ') }}
              </span>
            </div>
            <div class="sim-footer">
              <span class="sim-date">{{ formatDate(sim.created_at) }}</span>
              <button
                v-if="!sim.scored"
                class="resolve-btn"
                @click.stop="openResolveModal(sim)"
              >
                Resolve
              </button>
              <span v-else class="outcome-badge">
                {{ sim.actual_outcome }}
              </span>
            </div>
          </div>

          <div v-if="simulations.length === 0 && !loading" class="empty-state">
            <p>No simulations found.</p>
          </div>
        </div>
      </section>
    </main>

    <!-- Resolve Modal -->
    <div v-if="showResolveModal" class="modal-overlay" @click.self="closeResolveModal">
      <div class="modal-content">
        <h3 class="modal-title">Resolve Simulation Outcome</h3>
        <p class="modal-sim-id">{{ resolveTarget?.simulation_id }}</p>
        <p class="modal-question">{{ resolveTarget?.question }}</p>

        <div class="outcome-options">
          <label class="outcome-option">
            <input type="radio" v-model="resolveOutcome" value="bullish" />
            <span class="option-label bullish">Bullish</span>
          </label>
          <label class="outcome-option">
            <input type="radio" v-model="resolveOutcome" value="neutral" />
            <span class="option-label neutral">Neutral</span>
          </label>
          <label class="outcome-option">
            <input type="radio" v-model="resolveOutcome" value="bearish" />
            <span class="option-label bearish">Bearish</span>
          </label>
        </div>

        <div class="notes-input">
          <label>Notes (optional)</label>
          <textarea v-model="resolveNotes" placeholder="Add notes about the resolution..."></textarea>
        </div>

        <div class="modal-actions">
          <button class="cancel-btn" @click="closeResolveModal">Cancel</button>
          <button
            class="confirm-btn"
            @click="submitResolve"
            :disabled="!resolveOutcome || resolving"
          >
            {{ resolving ? 'Resolving...' : 'Confirm Resolution' }}
          </button>
        </div>
      </div>
    </div>

    <!-- Agent Detail Modal -->
    <div v-if="showAgentModal" class="modal-overlay" @click.self="closeAgentModal">
      <div class="modal-content agent-detail-modal">
        <div class="modal-header">
          <h3 class="modal-title">Agent Details</h3>
          <button class="close-btn" @click="closeAgentModal">&times;</button>
        </div>

        <div v-if="selectedAgent" class="agent-detail-content">
          <div class="agent-overview">
            <div class="agent-info">
              <h4 class="agent-detail-id">{{ selectedAgent.agent_id }}</h4>
              <p class="agent-detail-persona">{{ selectedAgent.persona }}</p>
            </div>
            <div class="agent-stats-grid">
              <div class="stat-item">
                <span class="stat-label">Hit Rate</span>
                <span class="stat-value">{{ (selectedAgent.overall_stats.hit_rate * 100).toFixed(1) }}%</span>
              </div>
              <div class="stat-item">
                <span class="stat-label">Alpha Score</span>
                <span class="stat-value" :class="{ positive: selectedAgent.overall_stats.alpha_score > 0 }">
                  {{ (selectedAgent.overall_stats.alpha_score * 100).toFixed(1) }}%
                </span>
              </div>
              <div class="stat-item">
                <span class="stat-label">Calibration</span>
                <span class="stat-value">{{ selectedAgent.overall_stats.calibration_score.toFixed(3) }}</span>
              </div>
              <div class="stat-item">
                <span class="stat-label">Total Sims</span>
                <span class="stat-value">{{ selectedAgent.overall_stats.total_simulations }}</span>
              </div>
            </div>
          </div>

          <div class="prediction-history">
            <h5>Prediction History</h5>
            <div class="history-list">
              <div
                v-for="sim in selectedAgent.simulations"
                :key="sim.simulation_id"
                class="history-item"
              >
                <div class="history-header">
                  <span class="history-sim-id">{{ sim.simulation_id.slice(0, 12) }}</span>
                  <span
                    class="history-result"
                    :class="{ correct: sim.correct === true, incorrect: sim.correct === false, pending: sim.correct === null }"
                  >
                    {{ sim.correct === null ? 'Pending' : (sim.correct ? 'Correct' : 'Incorrect') }}
                  </span>
                </div>
                <p class="history-prediction">
                  <strong>Predicted:</strong> {{ sim.predicted_outcome }} ({{ (sim.confidence * 100).toFixed(0) }}%)
                </p>
                <p v-if="sim.actual_outcome" class="history-actual">
                  <strong>Actual:</strong> {{ sim.actual_outcome }}
                </p>
                <span class="history-date">{{ formatDate(sim.created_at) }}</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()

// State
const loading = ref(false)
const stats = ref({
  total_agents: 0,
  total_simulations: 0,
  scored_simulations: 0,
  pending_simulations: 0,
  avg_hit_rate: 0,
  avg_calibration: 0,
  top_agent: null
})

const topAgents = ref([])
const simulations = ref([])
const availableTopics = ref([])

// Filters
const topicFilter = ref('')
const agentLimit = ref(20)
const simStatusFilter = ref('')

// Modals
const showResolveModal = ref(false)
const resolveTarget = ref(null)
const resolveOutcome = ref('')
const resolveNotes = ref('')
const resolving = ref(false)

const showAgentModal = ref(false)
const selectedAgent = ref(null)

// API base URL
const API_BASE = 'http://localhost:5001/api'

// API calls
const fetchStats = async () => {
  try {
    const res = await fetch(`${API_BASE}/ledger/stats`)
    const data = await res.json()
    if (data.success) {
      stats.value = data.data
    }
  } catch (err) {
    console.error('Failed to fetch stats:', err)
  }
}

const loadTopAgents = async () => {
  loading.value = true
  try {
    let url = `${API_BASE}/ledger/top?n=${agentLimit.value}`
    if (topicFilter.value) {
      url += `&topic=${encodeURIComponent(topicFilter.value)}`
    }
    const res = await fetch(url)
    const data = await res.json()
    if (data.success) {
      topAgents.value = data.data
      // Extract unique topics from agents
      const topics = new Set()
      data.data.forEach(agent => {
        agent.best_topics?.forEach(t => topics.add(t))
      })
      availableTopics.value = Array.from(topics).sort()
    }
  } catch (err) {
    console.error('Failed to load top agents:', err)
  } finally {
    loading.value = false
  }
}

const loadSimulations = async () => {
  try {
    let url = `${API_BASE}/ledger/simulations?limit=50`
    if (simStatusFilter.value) {
      url += `&status=${simStatusFilter.value}`
    }
    const res = await fetch(url)
    const data = await res.json()
    if (data.success) {
      simulations.value = data.data
    }
  } catch (err) {
    console.error('Failed to load simulations:', err)
  }
}

const refreshData = async () => {
  loading.value = true
  await Promise.all([
    fetchStats(),
    loadTopAgents(),
    loadSimulations()
  ])
  loading.value = false
}

// Agent detail
const showAgentDetail = async (agentId) => {
  try {
    const res = await fetch(`${API_BASE}/ledger/agents/${agentId}`)
    const data = await res.json()
    if (data.success) {
      selectedAgent.value = data.data
      showAgentModal.value = true
    }
  } catch (err) {
    console.error('Failed to fetch agent detail:', err)
  }
}

const closeAgentModal = () => {
  showAgentModal.value = false
  selectedAgent.value = null
}

// Resolve modal
const openResolveModal = (sim) => {
  resolveTarget.value = sim
  resolveOutcome.value = ''
  resolveNotes.value = ''
  showResolveModal.value = true
}

const closeResolveModal = () => {
  showResolveModal.value = false
  resolveTarget.value = null
}

const submitResolve = async () => {
  if (!resolveOutcome.value || !resolveTarget.value) return

  resolving.value = true
  try {
    const res = await fetch(`${API_BASE}/ledger/resolve`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        simulation_id: resolveTarget.value.simulation_id,
        actual_outcome: resolveOutcome.value,
        notes: resolveNotes.value
      })
    })
    const data = await res.json()
    if (data.success) {
      closeResolveModal()
      await refreshData()
    } else {
      alert('Failed to resolve: ' + (data.error || 'Unknown error'))
    }
  } catch (err) {
    console.error('Failed to resolve simulation:', err)
    alert('Failed to resolve simulation')
  } finally {
    resolving.value = false
  }
}

// Helpers
const getRankClass = (rank) => {
  if (rank === 1) return 'gold'
  if (rank === 2) return 'silver'
  if (rank === 3) return 'bronze'
  return ''
}

const truncatePersona = (persona) => {
  if (!persona) return 'N/A'
  return persona.length > 40 ? persona.slice(0, 40) + '...' : persona
}

const truncateQuestion = (question) => {
  if (!question) return 'N/A'
  return question.length > 100 ? question.slice(0, 100) + '...' : question
}

const formatDate = (dateStr) => {
  if (!dateStr) return ''
  const date = new Date(dateStr)
  return date.toLocaleDateString() + ' ' + date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
}

// Lifecycle
onMounted(() => {
  refreshData()
})
</script>

<style scoped>
.leaderboard-view {
  min-height: 100vh;
  background: #111;
  color: #fff;
  font-family: 'Space Grotesk', 'Noto Sans SC', system-ui, sans-serif;
}

/* Header */
.app-header {
  height: 60px;
  border-bottom: 1px solid #333;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 24px;
  background: #111;
}

.brand {
  font-family: 'JetBrains Mono', monospace;
  font-weight: 800;
  font-size: 18px;
  letter-spacing: 1px;
  cursor: pointer;
  color: #fff;
}

.page-title {
  font-size: 16px;
  font-weight: 600;
  color: #fff;
}

.refresh-btn {
  padding: 8px 16px;
  background: #222;
  border: 1px solid #444;
  color: #fff;
  border-radius: 6px;
  cursor: pointer;
  font-size: 13px;
  transition: all 0.2s;
}

.refresh-btn:hover:not(:disabled) {
  background: #333;
  border-color: #666;
}

.refresh-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Stats Section */
.stats-section {
  display: flex;
  gap: 16px;
  padding: 20px 24px;
  border-bottom: 1px solid #222;
  overflow-x: auto;
}

.stat-card {
  background: #1a1a1a;
  border: 1px solid #333;
  border-radius: 8px;
  padding: 16px 24px;
  min-width: 140px;
}

.stat-card.highlight {
  border-color: #FF4500;
}

.stat-card .stat-value {
  font-family: 'JetBrains Mono', monospace;
  font-size: 28px;
  font-weight: 700;
  color: #fff;
}

.stat-card .stat-value.top-agent {
  font-size: 14px;
}

.stat-card .stat-label {
  font-size: 12px;
  color: #888;
  margin-top: 4px;
}

/* Main Content */
.main-content {
  display: flex;
  gap: 24px;
  padding: 24px;
  min-height: calc(100vh - 180px);
}

/* Leaderboard Section */
.leaderboard-section {
  flex: 2;
  background: #1a1a1a;
  border: 1px solid #333;
  border-radius: 8px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  border-bottom: 1px solid #333;
}

.section-title {
  font-size: 16px;
  font-weight: 600;
  color: #fff;
}

.filter-controls {
  display: flex;
  gap: 8px;
}

.topic-select,
.status-select,
.limit-input {
  background: #222;
  border: 1px solid #444;
  color: #fff;
  padding: 6px 12px;
  border-radius: 4px;
  font-size: 12px;
}

.limit-input {
  width: 60px;
}

/* Table */
.table-container {
  flex: 1;
  overflow: auto;
}

.leaderboard-table {
  width: 100%;
  border-collapse: collapse;
}

.leaderboard-table th {
  text-align: left;
  padding: 12px 16px;
  background: #222;
  font-size: 11px;
  font-weight: 600;
  color: #888;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  position: sticky;
  top: 0;
  z-index: 10;
}

.leaderboard-table td {
  padding: 12px 16px;
  border-bottom: 1px solid #2a2a2a;
  font-size: 13px;
}

.agent-row {
  cursor: pointer;
  transition: background 0.2s;
}

.agent-row:hover {
  background: #222;
}

/* Rank Badge */
.rank-badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  font-family: 'JetBrains Mono', monospace;
  font-weight: 700;
  font-size: 12px;
  background: #333;
}

.rank-badge.gold {
  background: linear-gradient(135deg, #FFD700, #FFA500);
  color: #000;
}

.rank-badge.silver {
  background: linear-gradient(135deg, #C0C0C0, #808080);
  color: #000;
}

.rank-badge.bronze {
  background: linear-gradient(135deg, #CD7F32, #8B4513);
  color: #fff;
}

/* Metric Bar */
.metric-bar {
  display: flex;
  align-items: center;
  gap: 8px;
}

.bar-fill {
  height: 6px;
  border-radius: 3px;
  max-width: 60px;
}

.bar-fill.hit-rate {
  background: linear-gradient(90deg, #4CAF50, #8BC34A);
}

.metric-value {
  font-family: 'JetBrains Mono', monospace;
  font-size: 12px;
  color: #ccc;
}

/* Alpha Value */
.alpha-value {
  font-family: 'JetBrains Mono', monospace;
  font-weight: 600;
}

.alpha-value.positive {
  color: #4CAF50;
}

.alpha-value.negative {
  color: #F44336;
}

/* Topic Tags */
.topic-tags {
  display: flex;
  gap: 4px;
}

.topic-tag {
  padding: 2px 8px;
  background: #333;
  border-radius: 4px;
  font-size: 10px;
  color: #aaa;
}

/* Agent ID & Persona */
.agent-id {
  font-family: 'JetBrains Mono', monospace;
  font-size: 12px;
  color: #FF4500;
}

.persona-text {
  color: #888;
  font-size: 12px;
}

/* Simulations Section */
.simulations-section {
  flex: 1;
  background: #1a1a1a;
  border: 1px solid #333;
  border-radius: 8px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.simulations-list {
  flex: 1;
  overflow-y: auto;
  padding: 12px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.simulation-card {
  background: #222;
  border: 1px solid #333;
  border-radius: 8px;
  padding: 16px;
}

.sim-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.sim-id {
  font-family: 'JetBrains Mono', monospace;
  font-size: 12px;
  color: #888;
}

.sim-status {
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 10px;
  font-weight: 600;
}

.sim-status.scored {
  background: #1a3d1a;
  color: #4CAF50;
}

.sim-status.pending {
  background: #3d2a1a;
  color: #FF9800;
}

.sim-question {
  font-size: 13px;
  color: #ccc;
  margin-bottom: 8px;
  line-height: 1.4;
}

.sim-meta {
  display: flex;
  gap: 12px;
  font-size: 11px;
  color: #666;
  margin-bottom: 12px;
}

.sim-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.sim-date {
  font-size: 11px;
  color: #555;
}

.resolve-btn {
  padding: 4px 12px;
  background: #FF4500;
  border: none;
  color: #fff;
  border-radius: 4px;
  cursor: pointer;
  font-size: 11px;
  font-weight: 600;
}

.resolve-btn:hover {
  background: #FF5722;
}

.outcome-badge {
  padding: 4px 12px;
  background: #333;
  border-radius: 4px;
  font-size: 11px;
  color: #4CAF50;
  text-transform: capitalize;
}

/* Empty State */
.empty-state {
  text-align: center;
  padding: 40px;
  color: #666;
}

/* Modal */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background: #1a1a1a;
  border: 1px solid #333;
  border-radius: 12px;
  padding: 24px;
  max-width: 500px;
  width: 90%;
}

.agent-detail-modal {
  max-width: 700px;
  max-height: 80vh;
  overflow-y: auto;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.modal-title {
  font-size: 18px;
  font-weight: 600;
  color: #fff;
  margin-bottom: 16px;
}

.close-btn {
  background: none;
  border: none;
  color: #888;
  font-size: 24px;
  cursor: pointer;
}

.close-btn:hover {
  color: #fff;
}

.modal-sim-id {
  font-family: 'JetBrains Mono', monospace;
  font-size: 12px;
  color: #888;
  margin-bottom: 8px;
}

.modal-question {
  color: #ccc;
  margin-bottom: 20px;
  line-height: 1.5;
}

.outcome-options {
  display: flex;
  gap: 12px;
  margin-bottom: 20px;
}

.outcome-option {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 12px;
  background: #222;
  border: 1px solid #333;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
}

.outcome-option:has(input:checked) {
  border-color: #FF4500;
  background: #2a1a1a;
}

.outcome-option input {
  display: none;
}

.option-label {
  font-weight: 600;
}

.option-label.bullish {
  color: #4CAF50;
}

.option-label.neutral {
  color: #FFC107;
}

.option-label.bearish {
  color: #F44336;
}

.notes-input {
  margin-bottom: 20px;
}

.notes-input label {
  display: block;
  font-size: 12px;
  color: #888;
  margin-bottom: 8px;
}

.notes-input textarea {
  width: 100%;
  height: 80px;
  background: #222;
  border: 1px solid #333;
  border-radius: 8px;
  padding: 12px;
  color: #fff;
  font-size: 13px;
  resize: none;
}

.modal-actions {
  display: flex;
  gap: 12px;
  justify-content: flex-end;
}

.cancel-btn {
  padding: 10px 20px;
  background: #333;
  border: none;
  color: #fff;
  border-radius: 6px;
  cursor: pointer;
}

.confirm-btn {
  padding: 10px 20px;
  background: #FF4500;
  border: none;
  color: #fff;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 600;
}

.confirm-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Agent Detail Modal Content */
.agent-overview {
  margin-bottom: 24px;
}

.agent-detail-id {
  font-family: 'JetBrains Mono', monospace;
  font-size: 14px;
  color: #FF4500;
  margin-bottom: 4px;
}

.agent-detail-persona {
  color: #888;
  font-size: 13px;
  line-height: 1.4;
  margin-bottom: 16px;
}

.agent-stats-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 12px;
}

.stat-item {
  background: #222;
  border-radius: 8px;
  padding: 12px;
  text-align: center;
}

.stat-item .stat-label {
  font-size: 10px;
  color: #666;
  text-transform: uppercase;
  margin-bottom: 4px;
}

.stat-item .stat-value {
  font-family: 'JetBrains Mono', monospace;
  font-size: 18px;
  font-weight: 700;
}

.stat-item .stat-value.positive {
  color: #4CAF50;
}

.prediction-history h5 {
  font-size: 14px;
  color: #fff;
  margin-bottom: 12px;
}

.history-list {
  max-height: 300px;
  overflow-y: auto;
}

.history-item {
  background: #222;
  border-radius: 8px;
  padding: 12px;
  margin-bottom: 8px;
}

.history-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.history-sim-id {
  font-family: 'JetBrains Mono', monospace;
  font-size: 11px;
  color: #666;
}

.history-result {
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 10px;
  font-weight: 600;
}

.history-result.correct {
  background: #1a3d1a;
  color: #4CAF50;
}

.history-result.incorrect {
  background: #3d1a1a;
  color: #F44336;
}

.history-result.pending {
  background: #3d2a1a;
  color: #FF9800;
}

.history-prediction,
.history-actual {
  font-size: 12px;
  color: #aaa;
  margin-bottom: 4px;
}

.history-date {
  font-size: 10px;
  color: #555;
}

/* Responsive */
@media (max-width: 1024px) {
  .main-content {
    flex-direction: column;
  }

  .leaderboard-section,
  .simulations-section {
    flex: none;
  }

  .stats-section {
    flex-wrap: wrap;
  }

  .agent-stats-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}
</style>
