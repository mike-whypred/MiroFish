<template>
  <div class="brief-builder-container">
    <!-- Top Navigation Bar -->
    <nav class="navbar">
      <div class="nav-brand">
        <router-link to="/" class="brand-link">MIROFISH</router-link>
      </div>
      <div class="nav-links">
        <router-link to="/" class="nav-link">Home</router-link>
        <router-link to="/brief-builder" class="nav-link active">Brief Builder</router-link>
        <a href="https://github.com/666ghj/MiroFish" target="_blank" class="github-link">
          View on GitHub <span class="arrow">↗</span>
        </a>
      </div>
    </nav>

    <div class="main-layout">
      <!-- Main Content Area -->
      <div class="main-content">
        <!-- Header Section -->
        <header class="header-section">
          <h1 class="page-title">Brief Builder</h1>
          <p class="page-subtitle">Research any topic and generate a MiroFish-ready seed document</p>
          <span class="powered-badge">Powered by Brave Search + Perplexity + GPT-5</span>
        </header>

        <!-- Input Section -->
        <section class="input-section" v-if="!building && !result">
          <div class="input-card">
            <div class="input-header">
              <span class="input-label">>_ Topic / Question</span>
            </div>
            <textarea
              v-model="topic"
              class="topic-textarea"
              placeholder="Enter your topic or question... e.g. Will the Fed cut rates in Q2 2026?"
              rows="5"
            ></textarea>

            <div class="depth-section">
              <span class="depth-label">Research Depth</span>
              <div class="depth-pills">
                <button
                  v-for="d in depthOptions"
                  :key="d.value"
                  :class="['depth-pill', { active: depth === d.value }]"
                  @click="depth = d.value"
                >
                  <span class="depth-name">{{ d.name }}</span>
                  <span class="depth-meta">{{ d.meta }}</span>
                </button>
              </div>
            </div>

            <button
              class="build-btn"
              :disabled="!canBuild || building"
              @click="buildBrief"
            >
              <span>Build Brief</span>
              <span class="btn-arrow">→</span>
            </button>
          </div>
        </section>

        <!-- Progress Section -->
        <section class="progress-section" v-if="building">
          <div class="progress-card">
            <div class="step-pills">
              <div
                v-for="(step, idx) in steps"
                :key="step"
                :class="['step-pill', { active: currentStep === idx, completed: currentStep > idx }]"
              >
                <span class="step-dot"></span>
                <span class="step-text">{{ step }}</span>
              </div>
            </div>

            <div class="progress-message">
              <div class="spinner"></div>
              <span>{{ progressMessage }}</span>
            </div>

            <div class="topic-display">
              <span class="topic-label">Topic:</span>
              <span class="topic-text">{{ topic }}</span>
            </div>
          </div>
        </section>

        <!-- Result Section -->
        <section class="result-section" v-if="result">
          <!-- Executive Summary -->
          <div class="summary-card">
            <div class="summary-header">
              <span class="summary-icon">◈</span>
              <span class="summary-title">Executive Summary</span>
            </div>
            <div class="summary-content">{{ result.summary }}</div>
          </div>

          <!-- Three Column Row -->
          <div class="three-col-row">
            <div class="info-card">
              <div class="info-header">Key Facts</div>
              <ul class="info-list">
                <li v-for="(fact, idx) in result.key_facts" :key="idx">{{ fact }}</li>
              </ul>
              <div v-if="!result.key_facts?.length" class="empty-state">No key facts available</div>
            </div>

            <div class="info-card">
              <div class="info-header">Key Players</div>
              <ul class="info-list">
                <li v-for="(player, idx) in result.key_players" :key="idx">{{ player }}</li>
              </ul>
              <div v-if="!result.key_players?.length" class="empty-state">No key players identified</div>
            </div>

            <div class="info-card">
              <div class="info-header">Suggested Scenarios</div>
              <ul class="info-list">
                <li v-for="(scenario, idx) in result.suggested_scenarios" :key="idx">{{ scenario }}</li>
              </ul>
              <div v-if="!result.suggested_scenarios?.length" class="empty-state">No scenarios suggested</div>
            </div>
          </div>

          <!-- Seed Document (Collapsible) -->
          <details class="seed-document-details">
            <summary class="seed-summary">
              <span class="seed-icon">📄</span>
              <span>Full Seed Document</span>
              <span class="expand-hint">(click to expand)</span>
            </summary>
            <pre class="seed-content">{{ result.seed_document }}</pre>
          </details>

          <!-- Sources -->
          <div class="sources-card" v-if="result.sources?.length">
            <div class="sources-header">Sources ({{ Math.min(result.sources.length, 10) }} shown)</div>
            <ul class="sources-list">
              <li v-for="(source, idx) in result.sources.slice(0, 10)" :key="idx">
                <a :href="source.url" target="_blank" class="source-link">
                  {{ source.title || source.url }}
                  <span class="source-arrow">↗</span>
                </a>
              </li>
            </ul>
          </div>

          <!-- Action Buttons -->
          <div class="action-buttons">
            <button class="action-btn primary" @click="loadIntoMiroFish" :disabled="loadingAction">
              <span>🚀 Load into MiroFish</span>
            </button>
            <button class="action-btn secondary" @click="copySeedDocument">
              <span>📋 Copy Seed Document</span>
            </button>
            <button class="action-btn secondary" @click="resetForm">
              <span>🔄 Build Another</span>
            </button>
          </div>

          <div v-if="copySuccess" class="copy-toast">Copied to clipboard!</div>
        </section>

        <!-- Error Display -->
        <div v-if="error" class="error-card">
          <span class="error-icon">⚠</span>
          <span>{{ error }}</span>
          <button class="error-dismiss" @click="error = ''">×</button>
        </div>
      </div>

      <!-- History Sidebar -->
      <aside class="history-sidebar">
        <div class="sidebar-header">
          <span class="sidebar-title">Past Briefs</span>
        </div>

        <div v-if="historyLoading" class="sidebar-loading">
          <div class="spinner small"></div>
          <span>Loading...</span>
        </div>

        <div v-else-if="history.length === 0" class="sidebar-empty">
          <span class="empty-icon">📁</span>
          <span>No briefs yet</span>
        </div>

        <div v-else class="history-list">
          <div
            v-for="brief in history"
            :key="brief.brief_id"
            class="history-item"
            :class="{ active: result?.brief_id === brief.brief_id }"
            @click="loadBrief(brief.brief_id)"
          >
            <div class="history-topic">{{ truncate(brief.topic, 40) }}</div>
            <div class="history-meta">
              <span class="history-date">{{ formatDate(brief.created_at) }}</span>
              <span :class="['history-status', brief.status]">{{ brief.status }}</span>
            </div>
          </div>
        </div>
      </aside>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()

// State
const topic = ref('')
const depth = ref('standard')
const building = ref(false)
const result = ref(null)
const error = ref('')
const currentStep = ref(0)
const progressMessage = ref('')
const history = ref([])
const historyLoading = ref(false)
const loadingAction = ref(false)
const copySuccess = ref(false)

// Constants
const depthOptions = [
  { value: 'quick', name: 'Quick', meta: '3 dimensions ~2 min' },
  { value: 'standard', name: 'Standard', meta: '6 dimensions ~8 min' },
  { value: 'deep', name: 'Deep', meta: '10 dimensions ~15 min' }
]

const steps = ['Planning', 'Researching', 'Synthesising']

// Computed
const canBuild = computed(() => topic.value.trim().length > 0)

// Methods
const buildBrief = async () => {
  if (!canBuild.value || building.value) return

  building.value = true
  result.value = null
  error.value = ''
  currentStep.value = 0
  progressMessage.value = 'Initializing research pipeline...'

  try {
    // Start the build
    const startRes = await fetch('/api/brief/build', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ topic: topic.value, depth: depth.value })
    })

    const startData = await startRes.json()

    if (!startData.success) {
      throw new Error(startData.error || 'Failed to start brief building')
    }

    const briefId = startData.data.brief_id

    // Poll for status
    await pollForCompletion(briefId)

  } catch (e) {
    error.value = e.message || 'An error occurred'
    building.value = false
  }
}

const pollForCompletion = async (briefId) => {
  const dimensionCount = depth.value === 'quick' ? 3 : depth.value === 'standard' ? 6 : 10
  let pollCount = 0
  const maxPolls = 120 // 10 minutes max with 5s interval

  const poll = async () => {
    try {
      const res = await fetch(`/api/brief/${briefId}`)
      const data = await res.json()

      if (!data.success) {
        throw new Error(data.error || 'Failed to fetch brief status')
      }

      const brief = data.data

      // Update progress based on status
      if (brief.status === 'pending' || brief.status === 'planning') {
        currentStep.value = 0
        progressMessage.value = 'Planning research dimensions...'
      } else if (brief.status === 'researching') {
        currentStep.value = 1
        const currentDim = brief.current_dimension || 1
        const dimName = brief.current_dimension_name || 'data'
        progressMessage.value = `Researching dimension ${currentDim} of ${dimensionCount}: ${dimName}...`
      } else if (brief.status === 'synthesizing') {
        currentStep.value = 2
        progressMessage.value = 'Synthesising findings into coherent brief...'
      } else if (brief.status === 'completed') {
        result.value = brief
        building.value = false
        loadHistory() // Refresh history
        return
      } else if (brief.status === 'failed') {
        throw new Error(brief.error_message || 'Brief building failed')
      }

      pollCount++
      if (pollCount < maxPolls) {
        setTimeout(poll, 5000)
      } else {
        throw new Error('Brief building timed out')
      }

    } catch (e) {
      error.value = e.message || 'An error occurred'
      building.value = false
    }
  }

  await poll()
}

const loadBrief = async (briefId) => {
  try {
    const res = await fetch(`/api/brief/${briefId}`)
    const data = await res.json()

    if (!data.success) {
      throw new Error(data.error || 'Failed to load brief')
    }

    result.value = data.data
    topic.value = data.data.topic
    depth.value = data.data.depth
    building.value = false

  } catch (e) {
    error.value = e.message || 'Failed to load brief'
  }
}

const loadHistory = async () => {
  historyLoading.value = true
  try {
    const res = await fetch('/api/brief/list?limit=20')
    const data = await res.json()

    if (data.success) {
      history.value = data.data.briefs || []
    }
  } catch (e) {
    console.error('Failed to load history:', e)
  } finally {
    historyLoading.value = false
  }
}

const loadIntoMiroFish = async () => {
  if (!result.value?.brief_id || loadingAction.value) return

  loadingAction.value = true
  try {
    const res = await fetch(`/api/brief/${result.value.brief_id}/load-as-seed`, {
      method: 'POST'
    })
    const data = await res.json()

    if (!data.success) {
      throw new Error(data.error || 'Failed to load as seed')
    }

    // Store seed document for home page
    sessionStorage.setItem('mirofish_seed_document', data.data.seed_document)

    // Redirect to home
    router.push('/')

  } catch (e) {
    error.value = e.message || 'Failed to load into MiroFish'
  } finally {
    loadingAction.value = false
  }
}

const copySeedDocument = async () => {
  if (!result.value?.seed_document) return

  try {
    await navigator.clipboard.writeText(result.value.seed_document)
    copySuccess.value = true
    setTimeout(() => {
      copySuccess.value = false
    }, 2000)
  } catch (e) {
    error.value = 'Failed to copy to clipboard'
  }
}

const resetForm = () => {
  topic.value = ''
  depth.value = 'standard'
  result.value = null
  error.value = ''
  building.value = false
  currentStep.value = 0
}

const truncate = (text, length) => {
  if (!text) return ''
  return text.length > length ? text.substring(0, length) + '...' : text
}

const formatDate = (dateStr) => {
  if (!dateStr) return ''
  const date = new Date(dateStr)
  return date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' })
}

// Lifecycle
onMounted(() => {
  loadHistory()
})
</script>

<style scoped>
/* CSS Variables */
:root {
  --bg-dark: #0a0a0a;
  --bg-card: #111111;
  --bg-card-hover: #1a1a1a;
  --text-primary: #ffffff;
  --text-secondary: #888888;
  --text-muted: #555555;
  --accent: #FF4500;
  --accent-hover: #ff5722;
  --border: #222222;
  --border-light: #333333;
  --success: #22c55e;
  --warning: #f59e0b;
  --font-mono: 'JetBrains Mono', monospace;
  --font-sans: 'Space Grotesk', 'Noto Sans SC', system-ui, sans-serif;
}

.brief-builder-container {
  min-height: 100vh;
  background: #0a0a0a;
  color: #ffffff;
  font-family: var(--font-sans);
}

/* Navbar */
.navbar {
  height: 60px;
  background: #000000;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 40px;
  border-bottom: 1px solid #222222;
}

.nav-brand {
  font-family: var(--font-mono);
  font-weight: 800;
  letter-spacing: 1px;
  font-size: 1.2rem;
}

.brand-link {
  color: #ffffff;
  text-decoration: none;
}

.nav-links {
  display: flex;
  align-items: center;
  gap: 24px;
}

.nav-link {
  color: #888888;
  text-decoration: none;
  font-family: var(--font-mono);
  font-size: 0.85rem;
  transition: color 0.2s;
}

.nav-link:hover,
.nav-link.active {
  color: #ffffff;
}

.github-link {
  color: #888888;
  text-decoration: none;
  font-family: var(--font-mono);
  font-size: 0.85rem;
  display: flex;
  align-items: center;
  gap: 6px;
  transition: color 0.2s;
}

.github-link:hover {
  color: #ffffff;
}

.arrow {
  font-size: 0.75rem;
}

/* Main Layout */
.main-layout {
  display: flex;
  max-width: 1600px;
  margin: 0 auto;
  min-height: calc(100vh - 60px);
}

.main-content {
  flex: 1;
  padding: 40px 60px;
  max-width: calc(100% - 280px);
}

/* Header Section */
.header-section {
  margin-bottom: 40px;
}

.page-title {
  font-size: 2.5rem;
  font-weight: 600;
  margin: 0 0 8px 0;
  letter-spacing: -1px;
}

.page-subtitle {
  font-size: 1rem;
  color: #888888;
  margin: 0 0 16px 0;
}

.powered-badge {
  display: inline-block;
  font-family: var(--font-mono);
  font-size: 0.7rem;
  color: #555555;
  background: #1a1a1a;
  padding: 4px 10px;
  border-radius: 4px;
  border: 1px solid #333333;
}

/* Input Section */
.input-section {
  max-width: 800px;
}

.input-card {
  background: #111111;
  border: 1px solid #222222;
  padding: 24px;
}

.input-header {
  margin-bottom: 12px;
}

.input-label {
  font-family: var(--font-mono);
  font-size: 0.75rem;
  color: #666666;
}

.topic-textarea {
  width: 100%;
  background: #0a0a0a;
  border: 1px solid #333333;
  color: #ffffff;
  font-family: var(--font-mono);
  font-size: 0.95rem;
  padding: 16px;
  resize: vertical;
  min-height: 120px;
  outline: none;
  transition: border-color 0.2s;
}

.topic-textarea:focus {
  border-color: #FF4500;
}

.topic-textarea::placeholder {
  color: #555555;
}

.depth-section {
  margin-top: 24px;
}

.depth-label {
  display: block;
  font-family: var(--font-mono);
  font-size: 0.75rem;
  color: #666666;
  margin-bottom: 12px;
}

.depth-pills {
  display: flex;
  gap: 12px;
}

.depth-pill {
  flex: 1;
  background: #0a0a0a;
  border: 1px solid #333333;
  color: #888888;
  padding: 16px;
  cursor: pointer;
  transition: all 0.2s;
  text-align: left;
}

.depth-pill:hover {
  border-color: #555555;
  color: #ffffff;
}

.depth-pill.active {
  border-color: #FF4500;
  color: #ffffff;
  background: #1a1a1a;
}

.depth-name {
  display: block;
  font-weight: 600;
  font-size: 0.95rem;
  margin-bottom: 4px;
}

.depth-meta {
  display: block;
  font-family: var(--font-mono);
  font-size: 0.7rem;
  color: #666666;
}

.depth-pill.active .depth-meta {
  color: #FF4500;
}

.build-btn {
  width: 100%;
  margin-top: 24px;
  background: #FF4500;
  color: #ffffff;
  border: none;
  padding: 18px;
  font-family: var(--font-mono);
  font-weight: 700;
  font-size: 1rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  cursor: pointer;
  transition: all 0.2s;
  letter-spacing: 0.5px;
}

.build-btn:hover:not(:disabled) {
  background: #ff5722;
}

.build-btn:disabled {
  background: #333333;
  color: #666666;
  cursor: not-allowed;
}

.btn-arrow {
  font-size: 1.2rem;
}

/* Progress Section */
.progress-section {
  max-width: 800px;
}

.progress-card {
  background: #111111;
  border: 1px solid #222222;
  padding: 32px;
}

.step-pills {
  display: flex;
  gap: 16px;
  margin-bottom: 32px;
}

.step-pill {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 16px;
  background: #0a0a0a;
  border: 1px solid #333333;
  color: #555555;
  font-family: var(--font-mono);
  font-size: 0.8rem;
  transition: all 0.3s;
}

.step-pill.active {
  border-color: #FF4500;
  color: #ffffff;
  background: #1a1a1a;
}

.step-pill.completed {
  border-color: #22c55e;
  color: #22c55e;
}

.step-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: currentColor;
}

.step-pill.active .step-dot {
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.4; }
}

.progress-message {
  display: flex;
  align-items: center;
  gap: 16px;
  color: #ffffff;
  font-size: 1rem;
  margin-bottom: 24px;
}

.spinner {
  width: 24px;
  height: 24px;
  border: 2px solid #333333;
  border-top-color: #FF4500;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

.spinner.small {
  width: 16px;
  height: 16px;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.topic-display {
  padding-top: 16px;
  border-top: 1px solid #222222;
  font-family: var(--font-mono);
  font-size: 0.85rem;
}

.topic-label {
  color: #555555;
  margin-right: 8px;
}

.topic-text {
  color: #888888;
}

/* Result Section */
.result-section {
  max-width: 1000px;
}

.summary-card {
  background: linear-gradient(135deg, #1a1a1a 0%, #111111 100%);
  border: 1px solid #FF4500;
  padding: 24px;
  margin-bottom: 24px;
}

.summary-header {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 16px;
}

.summary-icon {
  color: #FF4500;
  font-size: 1.2rem;
}

.summary-title {
  font-family: var(--font-mono);
  font-size: 0.8rem;
  color: #FF4500;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.summary-content {
  font-size: 1rem;
  line-height: 1.7;
  color: #cccccc;
}

/* Three Column Row */
.three-col-row {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
  margin-bottom: 24px;
}

.info-card {
  background: #111111;
  border: 1px solid #222222;
  padding: 20px;
}

.info-header {
  font-family: var(--font-mono);
  font-size: 0.75rem;
  color: #666666;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 16px;
  padding-bottom: 8px;
  border-bottom: 1px solid #222222;
}

.info-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.info-list li {
  font-size: 0.9rem;
  color: #cccccc;
  padding: 8px 0;
  border-bottom: 1px solid #1a1a1a;
  line-height: 1.5;
}

.info-list li:last-child {
  border-bottom: none;
}

.empty-state {
  color: #555555;
  font-size: 0.85rem;
  font-style: italic;
}

/* Seed Document Details */
.seed-document-details {
  background: #111111;
  border: 1px solid #222222;
  margin-bottom: 24px;
}

.seed-summary {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 16px 20px;
  cursor: pointer;
  font-family: var(--font-mono);
  font-size: 0.9rem;
  color: #ffffff;
  user-select: none;
}

.seed-summary:hover {
  background: #1a1a1a;
}

.seed-icon {
  font-size: 1rem;
}

.expand-hint {
  color: #555555;
  font-size: 0.75rem;
  margin-left: auto;
}

.seed-content {
  padding: 20px;
  margin: 0;
  background: #0a0a0a;
  border-top: 1px solid #222222;
  font-family: var(--font-mono);
  font-size: 0.8rem;
  line-height: 1.6;
  color: #aaaaaa;
  white-space: pre-wrap;
  overflow-x: auto;
  max-height: 500px;
  overflow-y: auto;
}

/* Sources Card */
.sources-card {
  background: #111111;
  border: 1px solid #222222;
  padding: 20px;
  margin-bottom: 24px;
}

.sources-header {
  font-family: var(--font-mono);
  font-size: 0.75rem;
  color: #666666;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 16px;
}

.sources-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.sources-list li {
  padding: 8px 0;
  border-bottom: 1px solid #1a1a1a;
}

.sources-list li:last-child {
  border-bottom: none;
}

.source-link {
  color: #888888;
  text-decoration: none;
  font-size: 0.85rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  transition: color 0.2s;
}

.source-link:hover {
  color: #FF4500;
}

.source-arrow {
  font-size: 0.7rem;
}

/* Action Buttons */
.action-buttons {
  display: flex;
  gap: 12px;
}

.action-btn {
  flex: 1;
  padding: 16px;
  border: none;
  font-family: var(--font-mono);
  font-size: 0.85rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.action-btn.primary {
  background: #FF4500;
  color: #ffffff;
}

.action-btn.primary:hover:not(:disabled) {
  background: #ff5722;
}

.action-btn.primary:disabled {
  background: #333333;
  color: #666666;
  cursor: not-allowed;
}

.action-btn.secondary {
  background: #1a1a1a;
  color: #ffffff;
  border: 1px solid #333333;
}

.action-btn.secondary:hover {
  background: #222222;
  border-color: #444444;
}

.copy-toast {
  position: fixed;
  bottom: 40px;
  left: 50%;
  transform: translateX(-50%);
  background: #22c55e;
  color: #ffffff;
  padding: 12px 24px;
  font-family: var(--font-mono);
  font-size: 0.85rem;
  border-radius: 4px;
  animation: fadeInOut 2s ease-in-out;
}

@keyframes fadeInOut {
  0%, 100% { opacity: 0; }
  10%, 90% { opacity: 1; }
}

/* Error Card */
.error-card {
  display: flex;
  align-items: center;
  gap: 12px;
  background: #1a0a0a;
  border: 1px solid #ff4444;
  padding: 16px 20px;
  margin-top: 24px;
  color: #ff6666;
}

.error-icon {
  font-size: 1.2rem;
}

.error-dismiss {
  margin-left: auto;
  background: none;
  border: none;
  color: #ff6666;
  font-size: 1.2rem;
  cursor: pointer;
  padding: 0 4px;
}

.error-dismiss:hover {
  color: #ffffff;
}

/* History Sidebar */
.history-sidebar {
  width: 280px;
  background: #0d0d0d;
  border-left: 1px solid #222222;
  padding: 24px;
  min-height: calc(100vh - 60px);
}

.sidebar-header {
  margin-bottom: 20px;
  padding-bottom: 12px;
  border-bottom: 1px solid #222222;
}

.sidebar-title {
  font-family: var(--font-mono);
  font-size: 0.8rem;
  color: #666666;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.sidebar-loading {
  display: flex;
  align-items: center;
  gap: 12px;
  color: #555555;
  font-size: 0.85rem;
}

.sidebar-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  color: #444444;
  font-size: 0.85rem;
  padding: 40px 0;
}

.empty-icon {
  font-size: 2rem;
  opacity: 0.5;
}

.history-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.history-item {
  background: #111111;
  border: 1px solid #222222;
  padding: 12px;
  cursor: pointer;
  transition: all 0.2s;
}

.history-item:hover {
  background: #1a1a1a;
  border-color: #333333;
}

.history-item.active {
  border-color: #FF4500;
}

.history-topic {
  font-size: 0.85rem;
  color: #cccccc;
  margin-bottom: 8px;
  line-height: 1.4;
}

.history-meta {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-family: var(--font-mono);
  font-size: 0.7rem;
}

.history-date {
  color: #555555;
}

.history-status {
  padding: 2px 6px;
  border-radius: 2px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.history-status.completed {
  background: rgba(34, 197, 94, 0.2);
  color: #22c55e;
}

.history-status.pending,
.history-status.planning,
.history-status.researching,
.history-status.synthesizing {
  background: rgba(245, 158, 11, 0.2);
  color: #f59e0b;
}

.history-status.failed {
  background: rgba(255, 68, 68, 0.2);
  color: #ff4444;
}

/* Responsive */
@media (max-width: 1200px) {
  .three-col-row {
    grid-template-columns: 1fr;
  }

  .action-buttons {
    flex-direction: column;
  }
}

@media (max-width: 900px) {
  .main-layout {
    flex-direction: column-reverse;
  }

  .main-content {
    max-width: 100%;
    padding: 24px;
  }

  .history-sidebar {
    width: 100%;
    min-height: auto;
    border-left: none;
    border-bottom: 1px solid #222222;
  }

  .history-list {
    flex-direction: row;
    overflow-x: auto;
    gap: 12px;
    padding-bottom: 8px;
  }

  .history-item {
    min-width: 200px;
    flex-shrink: 0;
  }
}
</style>
