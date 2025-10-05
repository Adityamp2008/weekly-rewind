<script setup>
import { ref, onMounted, computed } from 'vue'
import { supabase } from './supabase'

// --- State Management ---
const newActivityName = ref('')
const selectedEnergyScore = ref(null)
const activities = ref([])
const isLoading = ref(true)

const energyOptions = [
  { score: 2, icon: 'fa-solid fa-face-laugh-beam', color: '#22c55e', label: 'Sangat Fun' },
  { score: 1, icon: 'fa-solid fa-face-smile', color: '#84cc16', label: 'Cukup Fun' },
  { score: 0, icon: 'fa-solid fa-face-meh', color: '#808080', label: 'Netral' },
  { score: -1, icon: 'fa-solid fa-face-frown', color: '#f97316', label: 'Melelahkan' },
  { score: -2, icon: 'fa-solid fa-face-tired', color: '#ef4444', label: 'Sangat Melelahkan' },
]

// --- Computed Properties untuk Papan Skor (FITUR BARU) ---
const dailyScore = computed(() => {
  return activities.value.reduce((total, activity) => total + activity.energy_score, 0)
})

const highestActivity = computed(() => {
  if (activities.value.length === 0) return null
  return [...activities.value].sort((a, b) => b.energy_score - a.energy_score)[0]
})

const lowestActivity = computed(() => {
  if (activities.value.length === 0) return null
  return [...activities.value].sort((a, b) => a.energy_score - b.energy_score)[0]
})

// --- Helper Function untuk Format Tanggal (FITUR BARU) ---
function formatDate(isoString) {
  const date = new Date(isoString)
  return date.toLocaleDateString('id-ID', {
    day: 'numeric',
    month: 'short',
    hour: '2-digit',
    minute: '2-digit',
  })
}

// --- API Functions ---
async function addActivity() {
  if (!newActivityName.value || selectedEnergyScore.value === null) return
  try {
    await supabase.from('activities').insert({ 
      name: newActivityName.value, 
      energy_score: selectedEnergyScore.value 
    })
    newActivityName.value = ''
    selectedEnergyScore.value = null
    fetchActivities()
  } catch (error) {
    console.error('Error adding activity:', error.message)
  }
}

async function fetchActivities() {
  try {
    // Ambil data 24 jam terakhir
    const oneDayAgo = new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString()
    const { data, error } = await supabase
      .from('activities')
      .select('*')
      .gte('created_at', oneDayAgo) // Filter data 24 jam terakhir
      .order('created_at', { ascending: false })
    if (error) throw error
    activities.value = data
  } catch (error) {
    console.error('Error fetching data:', error.message)
  } finally {
    isLoading.value = false
  }
}

// --- Lifecycle Hook ---
onMounted(() => {
  fetchActivities()
})
</script>

<template>
  <div class="app-container">
    <header class="app-header">
      <i class="fa-solid fa-chart-line header-icon"></i>
      <h1>Weekly Rewind</h1>
      <p>Bagaimana energimu hari ini?</p>
    </header>
    
    <main>
      <section class="card score-summary-card">
        <h2>Skor Hari Ini</h2>
        <div class="summary-grid">
          <div class="summary-item">
            <span class="label">Total Skor</span>
            <span class="value" :class="{ positive: dailyScore > 0, negative: dailyScore < 0 }">{{ dailyScore }}</span>
          </div>
          <div class="summary-item">
            <span class="label">Tertinggi</span>
            <span class="value icon" v-if="highestActivity">
              <i :class="energyOptions.find(o => o.score === highestActivity.energy_score)?.icon" :style="{ color: energyOptions.find(o => o.score === highestActivity.energy_score)?.color }"></i>
            </span>
            <span class="value" v-else>-</span>
          </div>
           <div class="summary-item">
            <span class="label">Terendah</span>
             <span class="value icon" v-if="lowestActivity">
              <i :class="energyOptions.find(o => o.score === lowestActivity.energy_score)?.icon" :style="{ color: energyOptions.find(o => o.score === lowestActivity.energy_score)?.color }"></i>
            </span>
            <span class="value" v-else>-</span>
          </div>
        </div>
      </section>

      <section class="card form-card">
        <form @submit.prevent="addActivity">
          <div class="input-group">
            <i class="fa-solid fa-pencil"></i>
            <input 
              type="text" 
              v-model="newActivityName" 
              placeholder="Tulis aktivitasmu di sini..."
            />
          </div>

          <div class="score-selector">
            <button 
              type="button" 
              v-for="option in energyOptions" 
              :key="option.score"
              class="score-button"
              :class="{ active: selectedEnergyScore === option.score }"
              @click="selectedEnergyScore = option.score"
              :style="{ '--active-color': option.color }"
            >
              <i :class="option.icon"></i>
            </button>
          </div>
          
          <button type="submit" class="submit-button" :disabled="!newActivityName || selectedEnergyScore === null">
            <i class="fa-solid fa-plus"></i> Tambah Aktivitas
          </button>
        </form>
      </section>

      <section class="activity-feed">
        <h2>Aktivitas Terakhir (24 Jam)</h2>
        <div v-if="isLoading" class="loading-state">
          <i class="fa-solid fa-spinner fa-spin"></i> Memuat...
        </div>
        <div v-else-if="activities.length === 0" class="empty-state">
          <i class="fa-solid fa-moon"></i>
          <p>Belum ada aktivitas.</p>
          <span>Mulai catat untuk melihat riwayatmu.</span>
        </div>
        <ul v-else class="activity-list">
          <li v-for="activity in activities" :key="activity.id" class="card activity-item">
            <div class="activity-details">
              <span class="activity-name">{{ activity.name }}</span>
              <span class="activity-date">{{ formatDate(activity.created_at) }}</span>
            </div>
            <span class="activity-score" :style="{ color: energyOptions.find(o => o.score === activity.energy_score)?.color }">
              <i :class="energyOptions.find(o => o.score === activity.energy_score)?.icon"></i>
            </span>
          </li>
        </ul>
      </section>
    </main>
  </div>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');

:root {
  --bg-primary: #0f0f0f;
  --bg-secondary: #1a1a1a;
  --surface: #242424;
  --primary: #3b82f6;
  --primary-hover: #2563eb;
  --text-primary: #f5f5f5;
  --text-secondary: #a3a3a3;
  --border: #2a2a2a;
  --green: #10b981;
  --red: #ef4444;
  --shadow: rgba(0, 0, 0, 0.3);
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  -webkit-tap-highlight-color: transparent;
}

body {
  font-family: 'Inter', -apple-system, sans-serif;
  background: var(--bg-primary);
  color: var(--text-primary);
  line-height: 1.6;
  min-height: 100vh;
}

.app-container {
  max-width: 420px;
  margin: 0 auto;
  padding: 1rem;
  padding-bottom: 2rem;
}

/* Header */
.app-header {
  text-align: center;
  padding: 1.5rem 0 2rem;
}

.header-icon {
  font-size: 2.5rem;
  background: linear-gradient(135deg, var(--primary), #8b5cf6);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  margin-bottom: 0.75rem;
}

.app-header h1 {
  font-size: 1.75rem;
  font-weight: 700;
  margin-bottom: 0.5rem;
  letter-spacing: -0.02em;
}

.app-header p {
  color: var(--text-secondary);
  font-size: 0.95rem;
}

/* Card Base */
.card {
  background: var(--bg-secondary);
  border-radius: 20px;
  padding: 1.25rem;
  border: 1px solid var(--border);
  margin-bottom: 1.5rem;
  box-shadow: 0 4px 12px var(--shadow);
}

/* Score Summary */
.score-summary-card h2 {
  font-size: 1.1rem;
  font-weight: 600;
  margin-bottom: 1.25rem;
  text-align: center;
  color: var(--text-secondary);
  text-transform: uppercase;
  letter-spacing: 0.05em;
  font-size: 0.75rem;
}

.summary-grid {
  display: flex;
  justify-content: space-around;
  gap: 0.5rem;
}

.summary-item {
  text-align: center;
  flex: 1;
}

.summary-item .label {
  display: block;
  font-size: 0.75rem;
  color: var(--text-secondary);
  margin-bottom: 0.5rem;
  font-weight: 500;
}

.summary-item .value {
  font-size: 2rem;
  font-weight: 700;
  display: block;
}

.summary-item .value.icon {
  font-size: 2.25rem;
}

.summary-item .value.positive {
  color: var(--green);
}

.summary-item .value.negative {
  color: var(--red);
}

/* Form */
.input-group {
  display: flex;
  align-items: center;
  background: var(--surface);
  border-radius: 14px;
  padding: 0 1rem;
  margin-bottom: 1.25rem;
  border: 2px solid transparent;
  transition: all 0.2s ease;
}

.input-group:focus-within {
  border-color: var(--primary);
  background: var(--bg-secondary);
}

.input-group i {
  color: var(--text-secondary);
  margin-right: 0.75rem;
  font-size: 1rem;
}

.input-group input {
  flex: 1;
  background: none;
  border: none;
  outline: none;
  padding: 1rem 0;
  color: var(--text-primary);
  font-size: 0.95rem;
  font-family: 'Inter', sans-serif;
}

.input-group input::placeholder {
  color: var(--text-secondary);
}

/* Score Selector */
.score-selector {
  display: flex;
  justify-content: space-between;
  gap: 0.5rem;
  margin-bottom: 1.25rem;
}

.score-button {
  background: var(--surface);
  border: 2px solid transparent;
  color: var(--text-secondary);
  font-size: 1.5rem;
  cursor: pointer;
  padding: 0;
  border-radius: 12px;
  width: 100%;
  aspect-ratio: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
}

.score-button:active {
  transform: scale(0.95);
}

.score-button.active {
  color: var(--active-color);
  border-color: var(--active-color);
  background: var(--bg-secondary);
  transform: scale(1.05);
}

/* Submit Button */
.submit-button {
  width: 100%;
  padding: 1rem;
  font-size: 0.95rem;
  font-weight: 600;
  color: white;
  background: var(--primary);
  border: none;
  border-radius: 14px;
  cursor: pointer;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 0.5rem;
  transition: all 0.2s ease;
  box-shadow: 0 4px 12px rgba(59, 130, 246, 0.3);
}

.submit-button:active {
  transform: scale(0.98);
}

.submit-button:disabled {
  background: var(--surface);
  color: var(--text-secondary);
  cursor: not-allowed;
  box-shadow: none;
}

/* Activity Feed */
.activity-feed h2 {
  font-size: 1.1rem;
  font-weight: 600;
  margin-bottom: 1rem;
  color: var(--text-secondary);
  text-transform: uppercase;
  letter-spacing: 0.05em;
  font-size: 0.75rem;
  padding-left: 0.25rem;
}

.loading-state,
.empty-state {
  text-align: center;
  padding: 3rem 1rem;
  color: var(--text-secondary);
}

.loading-state i,
.empty-state i {
  font-size: 2.5rem;
  margin-bottom: 1rem;
  opacity: 0.5;
}

.empty-state p {
  font-weight: 500;
  margin-bottom: 0.25rem;
  color: var(--text-primary);
}

.empty-state span {
  font-size: 0.9rem;
}

/* Activity List */
.activity-list {
  list-style: none;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.activity-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  transition: all 0.2s ease;
  margin-bottom: 0;
}

.activity-item:active {
  transform: scale(0.98);
}

.activity-details {
  flex: 1;
  min-width: 0;
}

.activity-name {
  font-size: 0.95rem;
  font-weight: 500;
  display: block;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  margin-bottom: 0.25rem;
}

.activity-date {
  font-size: 0.75rem;
  color: var(--text-secondary);
}

.activity-score {
  font-size: 1.75rem;
  margin-left: 1rem;
  flex-shrink: 0;
}

/* Smooth Animations */
@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.card {
  animation: fadeIn 0.3s ease;
}

/* Mobile Optimization */
@media (max-width: 380px) {
  .app-container {
    padding: 0.75rem;
  }
  
  .card {
    padding: 1rem;
  }
  
  .score-button {
    font-size: 1.25rem;
  }
  
  .summary-item .value {
    font-size: 1.75rem;
  }
}
</style>