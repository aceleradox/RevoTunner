<template>
  <div class="app" :style="{ background: bgBackground }">

    <!-- PLAYER + DISCO -->
    <div class="player-area">
      <div
        class="dj-disk"
        :class="{ spinning: isPlaying }"
        @click="rewind"
        title="Voltar para o início"
      ></div>

      <iframe
        ref="player"
        class="player"
        :src="embedUrl"
        frameborder="0"
        allow="autoplay; encrypted-media"
      ></iframe>
    </div>

    <!-- AUTO PAD -->
    <div class="auto-panel">
      <input
        v-model="autoSequence"
        placeholder="Sequência de pads (ex: 1,5,20,60)"
      />
      <input
        type="number"
        min="1"
        v-model.number="autoInterval"
        placeholder="Intervalo (s)"
      />
      <button @click="toggleAutoPads">
        {{ autoRunning ? '⏹ Parar' : '▶ Auto Pads' }}
      </button>
    </div>

    <!-- CONTROLES -->
    <div class="controls">
      <button @click="togglePlay">▶ / ⏸</button>
      <button @click="nextMusic">⏭ Próxima</button>
    </div>

    <!-- MUSIC SLOTS -->
    <div class="slots">
      <div class="slot" v-for="(m, i) in musics" :key="i">
        <input v-model="m.url" placeholder="Link completo do YouTube" />
        <button @click="playMusic(i)">▶</button>
      </div>
    </div>

    <!-- LAUNCH PAD 120 -->
    <div class="pads">
      <button
        v-for="(sec, i) in padTimes"
        :key="i"
        class="pad"
        @click="padClick(i, sec)"
      >
        {{ i + 1 }}
      </button>
    </div>

  </div>
</template>

<script setup>
import { ref, computed, onUnmounted } from 'vue'

/* =========================
   STATE
========================= */
const player = ref(null)
const currentIndex = ref(0)
const isPlaying = ref(true)

const bgBackground = ref('#000')

/* =========================
   MUSIC LIST
========================= */
const musics = ref([
  { url: 'https://www.youtube.com/watch?v=dQw4w9WgXcQ' },
  { url: 'https://www.youtube.com/watch?v=3tmd-ClpJxA' },
  { url: '' }
])

/* =========================
   PADS CONFIG
========================= */
const PAD_COUNT = 120
const TOTAL_SECONDS = 20 * 60 // 20 minutos
const STEP = TOTAL_SECONDS / PAD_COUNT

const padTimes = Array.from({ length: PAD_COUNT }, (_, i) =>
  Math.floor(i * STEP)
)

/* =========================
   COLORS
========================= */
const colors = [
  '#ff7a00','#22c55e','#3b82f6','#a855f7',
  '#ec4899','#facc15','#14b8a6','#ef4444',
  '#6366f1','#84cc16','#06b6d4','#f97316',
  '#d946ef','#10b981','#eab308','#0ea5e9'
]

/* =========================
   AUTO PAD
========================= */
const autoSequence = ref('')
const autoInterval = ref(1)
const autoRunning = ref(false)

let autoTimer = null
let autoPos = 0

/* =========================
   YOUTUBE API
========================= */
function extractId(url) {
  const m = url.match(/(v=|youtu\.be\/)([\w-]+)/)
  return m ? m[2] : ''
}

const embedUrl = computed(() => {
  const id = extractId(musics.value[currentIndex.value].url)
  return `https://www.youtube.com/embed/${id}?enablejsapi=1&autoplay=1`
})

function post(cmd, args = []) {
  player.value?.contentWindow.postMessage(
    JSON.stringify({ event: 'command', func: cmd, args }),
    '*'
  )
}

/* =========================
   CONTROLS
========================= */
function togglePlay() {
  isPlaying.value ? post('pauseVideo') : post('playVideo')
  isPlaying.value = !isPlaying.value
}

function playMusic(i) {
  currentIndex.value = i
  isPlaying.value = true
}

function nextMusic() {
  currentIndex.value = (currentIndex.value + 1) % musics.value.length
  isPlaying.value = true
}

function rewind() {
  post('seekTo', [0, true])
}

/* =========================
   PAD CLICK
========================= */
function padClick(index, sec) {
  post('seekTo', [sec, true])

  const c = colors[Math.floor(Math.random() * colors.length)]
  bgBackground.value = `radial-gradient(circle, ${c}, #000)`

  setTimeout(() => {
    bgBackground.value = '#000'
  }, 200)
}

/* =========================
   AUTO PAD ENGINE
========================= */
function toggleAutoPads() {
  autoRunning.value = !autoRunning.value

  if (!autoRunning.value) {
    clearInterval(autoTimer)
    return
  }

  const sequence = autoSequence.value
    .split(',')
    .map(n => parseInt(n.trim()) - 1)
    .filter(n => n >= 0 && n < PAD_COUNT)

  autoPos = 0

  autoTimer = setInterval(() => {
    if (!sequence.length) return
    const pad = sequence[autoPos % sequence.length]
    padClick(pad, padTimes[pad])
    autoPos++
  }, autoInterval.value * 1000)
}

onUnmounted(() => clearInterval(autoTimer))
</script>

<style scoped>
.app {
  min-height: 100vh;
  padding: 18px;
  color: #fff;
  transition: background 0.2s ease;
  background: radial-gradient(circle, #000, #111);
  font-family: system-ui, sans-serif;
}

/* PLAYER */
.player-area {
  display: flex;
  gap: 18px;
  align-items: center;
}

.player {
  flex: 1;
  height: 360px;
  border-radius: 18px;
  border: 1px solid #222;
}

/* DISK */
.dj-disk {
  width: 120px;
  height: 120px;
  border-radius: 50%;
  border: 6px solid #ff7a00;
  cursor: pointer;
  background:
    radial-gradient(circle, #000 30%, #222 60%, #000);
}

.spinning {
  animation: spin 1.9s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg) }
  to { transform: rotate(360deg) }
}

/* AUTO PANEL */
.auto-panel {
  display: flex;
  gap: 8px;
  margin: 14px 0;
}

.auto-panel input {
  padding: 7px;
  border-radius: 6px;
  border: none;
}

.auto-panel button {
  background: #ff7a00;
  border: none;
  padding: 8px 14px;
  border-radius: 8px;
  font-weight: bold;
  cursor: pointer;
}

/* CONTROLS */
.controls button {
  background: #ff7a00;
  border: none;
  padding: 10px 16px;
  border-radius: 10px;
  margin: 6px;
  font-weight: bold;
  cursor: pointer;
}

/* SLOTS */
.slots {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
  margin-top: 10px;
}

.slot {
  background: #111;
  padding: 10px;
  border-radius: 10px;
}

.slot input {
  width: 100%;
  margin-bottom: 6px;
}

/* PADS */
.pads {
  margin-top: 20px;
  display: grid;
  grid-template-columns: repeat(12, 1fr);
  gap: 6px;
}

.pad {
  height: 28px;
  border-radius: 6px;
  font-size: 10px;
  font-weight: bold;
  cursor: pointer;
  border: none;
  background: linear-gradient(135deg, #333, #111);
  color: #fff;
}

.pad:active {
  transform: scale(0.92);
}
</style>
