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
              <br>
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

/* --- 🎨 SISTEM DESAIN & VARIABLES --- */
:root {
  /* Colors */
  --c-bg: #0A0A0A;
  --c-surface-1: #141414;
  --c-surface-2: #1F1F1F;
  --c-surface-3: #2A2A2A;
  --c-primary: #007AFF;
  --c-primary-hover: #3395FF;
  --c-text-1: #F5F5F7;
  --c-text-2: #8A8A8E;
  --c-border: #2D2D2D;
  --c-green: #34D399;
  --c-red: #F87171;

  /* Typography */
  --font-sans: 'Inter', -apple-system, sans-serif;
  --fs-xs: 0.75rem;   /* 12px */
  --fs-sm: 0.875rem;  /* 14px */
  --fs-base: 1rem;    /* 16px */
  --fs-lg: 1.125rem;  /* 18px */
  --fs-xl: 1.25rem;   /* 20px */
  --fs-2xl: 1.5rem;   /* 24px */
  --fs-3xl: 2rem;     /* 32px */

  /* Spacing */
  --space-1: 0.25rem; --space-2: 0.5rem;  --space-3: 0.75rem;
  --space-4: 1rem;    --space-5: 1.25rem; --space-6: 1.5rem;

  /* Radius & Transitions */
  --radius-md: 12px;
  --radius-lg: 16px;
  --transition: 0.2s cubic-bezier(0.25, 0.46, 0.45, 0.94);
}

/* --- 🌐 GLOBAL & RESET --- */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  -webkit-tap-highlight-color: transparent;
}

body {
  font-family: var(--font-sans);
  background: var(--c-bg);
  color: var(--c-text-1);
  line-height: 1.6;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.app-container {
  max-width: 480px;
  margin: 0 auto;
  padding: var(--space-4);
}

/* --- 🏠 HEADER --- */
.app-header {
  text-align: center;
  margin-bottom: var(--space-6);
}

.header-icon {
  font-size: var(--fs-3xl);
  color: var(--c-primary);
  margin-bottom: var(--space-3);
}

.app-header h1 {
  font-size: var(--fs-2xl);
  font-weight: 700;
  margin-bottom: var(--space-1);
}

.app-header p {
  color: var(--c-text-2);
  font-size: var(--fs-base);
}

/* --- 🃏 CARD BASE --- */
.card {
  background: var(--c-surface-1);
  border-radius: var(--radius-lg);
  padding: var(--space-5);
  border: 1px solid var(--c-border);
  margin-bottom: var(--space-6);
  transition: transform var(--transition), box-shadow var(--transition);
}

/* --- 📊 SCORE SUMMARY --- */
.score-summary-card h2 {
  font-size: var(--fs-xs);
  font-weight: 600;
  margin-bottom: var(--space-5);
  text-align: center;
  color: var(--c-text-2);
  text-transform: uppercase;
  letter-spacing: 0.075em;
}

.summary-grid {
  display: flex;
  justify-content: space-around;
  gap: var(--space-2);
}

.summary-item {
  text-align: center;
  flex: 1;
}

.summary-item .label {
  display: block;
  font-size: var(--fs-xs);
  color: var(--c-text-2);
  margin-bottom: var(--space-2);
  font-weight: 500;
}

.summary-item .value {
  font-size: var(--fs-3xl);
  font-weight: 700;
  display: block;
  line-height: 1;
}

.summary-item .value.icon {
  font-size: 2.5rem;
}

.summary-item .value.positive { color: var(--c-green); }
.summary-item .value.negative { color: var(--c-red); }

/* --- ✍️ FORM --- */
.input-group {
  display: flex;
  align-items: center;
  background: var(--c-surface-2);
  border-radius: var(--radius-md);
  padding: 0 var(--space-4);
  margin-bottom: var(--space-5);
  border: 1px solid var(--c-border);
  transition: border-color var(--transition), background-color var(--transition);
}

.input-group:focus-within {
  border-color: var(--c-primary);
  background: var(--c-surface-1);
}

.input-group i {
  color: var(--c-text-2);
  margin-right: var(--space-3);
  font-size: var(--fs-base);
}

.input-group input {
  flex: 1;
  background: none;
  border: none;
  outline: none;
  padding: var(--space-4) 0;
  color: var(--c-text-1);
  font-size: var(--fs-base);
  font-family: var(--font-sans);
}

.input-group input::placeholder { color: var(--c-text-2); }

/* Score Selector */
.score-selector {
  display: flex;
  gap: var(--space-3);
  margin-bottom: var(--space-5);
}

.score-button {
  background: var(--c-surface-2);
  border: 1px solid var(--c-border);
  color: var(--c-text-2);
  font-size: var(--fs-2xl);
  cursor: pointer;
  padding: 0;
  border-radius: var(--radius-md);
  flex: 1;
  aspect-ratio: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all var(--transition);
}

.score-button:hover {
  background: var(--c-surface-3);
  transform: translateY(-2px);
}
.score-button:active { transform: scale(0.95); }

.score-button.active {
  color: var(--active-color);
  border-color: var(--active-color);
  background: var(--c-surface-1);
  box-shadow: 0 0 15px -2px var(--active-color, var(--c-primary));
}

/* Submit Button */
.submit-button {
  width: 100%;
  padding: var(--space-4);
  font-size: var(--fs-base);
  font-weight: 600;
  color: white;
  background: var(--c-primary);
  border: none;
  border-radius: var(--radius-md);
  cursor: pointer;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: var(--space-3);
  transition: all var(--transition);
}

.submit-button:hover:not(:disabled) {
  background: var(--c-primary-hover);
  transform: translateY(-2px);
}
.submit-button:active:not(:disabled) {
  transform: scale(0.98) translateY(0);
}

.submit-button:disabled {
  background: var(--c-surface-2);
  color: var(--c-text-2);
  cursor: not-allowed;
}

/* --- 📜 ACTIVITY FEED --- */
.activity-feed h2 {
  font-size: var(--fs-xs);
  font-weight: 600;
  margin-bottom: var(--space-4);
  color: var(--c-text-2);
  text-transform: uppercase;
  letter-spacing: 0.075em;
  padding-left: var(--space-2);
}

.loading-state, .empty-state {
  text-align: center;
  padding: 3rem var(--space-4);
  color: var(--c-text-2);
}

.loading-state i, .empty-state i {
  font-size: 2.5rem;
  margin-bottom: var(--space-4);
  opacity: 0.5;
}

.empty-state p {
  font-weight: 500;
  margin-bottom: var(--space-1);
  color: var(--c-text-1);
}

/* Activity List & Staggered Animation */
.activity-list {
  list-style: none;
  display: flex;
  flex-direction: column;
  gap: var(--space-3);
}

.activity-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: var(--space-4);
  transition: all var(--transition);
  margin-bottom: 0;
  opacity: 0; /* Start hidden for animation */
  animation: fadeInUp 0.5s ease forwards;
}

/* Delay animasi untuk beberapa item pertama agar terlihat berurutan */
.activity-item:nth-child(1) { animation-delay: 0.1s; }
.activity-item:nth-child(2) { animation-delay: 0.2s; }
.activity-item:nth-child(3) { animation-delay: 0.3s; }
.activity-item:nth-child(4) { animation-delay: 0.4s; }
.activity-item:nth-child(5) { animation-delay: 0.5s; }

.activity-item:hover {
  transform: scale(1.02);
  background: var(--c-surface-2);
}

.activity-details { flex: 1; min-width: 0; }

.activity-name {
  font-size: var(--fs-sm);
  font-weight: 500;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  margin-bottom: var(--space-1);
}

.activity-date {
  font-size: var(--fs-xs);
  color: var(--c-text-2);
}

.activity-score {
  font-size: var(--fs-2xl);
  margin-left: var(--space-4);
  flex-shrink: 0;
}

/* --- ✨ ANIMATIONS --- */
@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* --- 📱 MOBILE OPTIMIZATION --- */
@media (max-width: 380px) {
  .app-container {
    padding: var(--space-3);
  }
  .card {
    padding: var(--space-4);
  }
  .summary-item .value {
    font-size: var(--fs-2xl);
  }
  .score-button {
    font-size: var(--fs-xl);
  }
}

</style>