<template>
  <div class="card timer-dashboard">
    <div class="card-header">
      <h2>⏱️ 小組報告計時器</h2>
    </div>

    <div class="display-container" :class="{ 'time-up': isFinished, 'warning': timeLeft <= 10 && timeLeft > 0 }">
      <transition name="bounce">
        <div v-if="isFinished" class="finish-banner">報告結束！</div>
      </transition>

      <div class="timer-digits">
        {{ formattedTime }}
      </div>
    </div>

    <div class="time-setup-section">
      <div class="preset-row">
        <button class="outline" @click="setTime(3)">3 分鐘</button>
        <button class="outline" @click="setTime(5)">5 分鐘</button>
        <button class="outline" @click="setTime(10)">10 分鐘</button>
        <button class="outline" @click="setTime(15)">15 分鐘</button>
      </div>

      <div class="custom-time-row">
        <span>自訂時間：</span>
        <input 
          type="number" 
          v-model.number="customMinutes" 
          min="0" 
          max="99" 
          class="time-input"
          placeholder="0"
        > 
        <span>分</span>
        <input 
          type="number" 
          v-model.number="customSeconds" 
          min="0" 
          max="59" 
          class="time-input"
          placeholder="0"
        > 
        <span>秒</span>
        <button class="set-custom-btn" @click="applyCustomTime">✔ 設定</button>
      </div>
    </div>

    <div class="controls-row">
      <button v-if="!isRunning" class="start-btn" @click="startTimer" :disabled="timeLeft === 0">▶ 開始計時</button>
      <button v-else class="pause-btn" @click="pauseTimer">⏸ 暫停</button>
      <button class="outline reset-btn" @click="resetTimer">↺ 重置</button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onUnmounted } from 'vue'

const timeLeft = ref(180) 
const isRunning = ref(false)
const isFinished = ref(false)
let timerInterval = null
let initialTime = 180

const customMinutes = ref('')
const customSeconds = ref('')

// 🎵 載入音效：結束響鈴聲 (傳統機械鬧鐘的劇烈響鈴聲)
const alarmSound = new Audio('https://actions.google.com/sounds/v1/alarms/mechanical_clock_ring.ogg')

const formattedTime = computed(() => {
  const m = Math.floor(timeLeft.value / 60).toString().padStart(2, '0')
  const s = (timeLeft.value % 60).toString().padStart(2, '0')
  return `${m}:${s}`
})

const setTime = (minutes) => {
  initialTime = minutes * 60
  resetTimer()
  customMinutes.value = ''
  customSeconds.value = ''
}

const applyCustomTime = () => {
  const m = Math.max(0, parseInt(customMinutes.value) || 0)
  const s = Math.max(0, parseInt(customSeconds.value) || 0)
  const totalSeconds = (m * 60) + s
  
  if (totalSeconds > 0) {
    initialTime = totalSeconds
    resetTimer()
  } else {
    alert('請輸入大於 0 的時間！')
  }
}

const startTimer = () => {
  if (isRunning.value || timeLeft.value === 0) return
  
  isFinished.value = false
  alarmSound.pause()
  alarmSound.currentTime = 0

  isRunning.value = true
  timerInterval = setInterval(() => {
    if (timeLeft.value > 0) {
      timeLeft.value--
      // 倒數音效已移除，僅保留最後 10 秒的視覺閃爍效果 (CSS .warning)
    } else {
      handleTimeUp()
    }
  }, 1000)
}

const handleTimeUp = () => {
  pauseTimer()
  isFinished.value = true
  // 🔊 播放機械鬧鐘響鈴
  alarmSound.play().catch(err => console.log('音效播放被阻擋:', err))
}

const pauseTimer = () => {
  isRunning.value = false
  clearInterval(timerInterval)
}

const resetTimer = () => {
  pauseTimer()
  timeLeft.value = initialTime
  isFinished.value = false
  // 重置時切斷鬧鐘聲音
  alarmSound.pause()
  alarmSound.currentTime = 0
}

onUnmounted(() => {
  clearInterval(timerInterval)
  alarmSound.pause()
})
</script>

<style scoped>
.timer-dashboard {
  display: flex;
  flex-direction: column;
  align-items: center;
  max-width: 800px;
  margin: 0 auto;
  text-align: center;
}

.card-header h2 { margin-top: 0; }

/* === 主顯示區 === */
.display-container {
  width: 100%;
  background: var(--color-1);
  border: 6px solid var(--color-5);
  padding: 40px 20px;
  margin: 10px 0 20px 0;
  box-shadow: inset 5px 5px 15px rgba(0,0,0,0.1), 8px 8px 0px var(--color-4);
  position: relative;
  min-height: 220px;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.timer-digits {
  font-size: 8rem;
  font-weight: bold;
  color: var(--color-5);
  text-shadow: 4px 4px 0px var(--color-3);
  font-variant-numeric: tabular-nums;
}

/* 剩餘 10 秒閃爍提示 */
.warning .timer-digits {
  color: #ef4444;
  animation: blink 1s infinite;
}

/* 時間結束樣式 */
.time-up {
  border-color: #ef4444;
  background-color: #fee2e2;
}

.finish-banner {
  position: absolute;
  top: -20px;
  left: 50%;
  transform: translateX(-50%);
  background: #ef4444;
  color: white;
  padding: 10px 30px;
  border: 4px solid var(--color-5);
  font-size: 2rem;
  z-index: 20;
  box-shadow: 4px 4px 0px var(--color-5);
}

/* === 設定時間區塊 === */
.time-setup-section {
  width: 100%;
  background: var(--color-2);
  border: 4px dashed var(--color-4);
  padding: 20px;
  margin-bottom: 30px;
}

.preset-row {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
  flex-wrap: wrap;
  justify-content: center;
}

.custom-time-row {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  font-size: 1.2rem;
  font-weight: bold;
  color: var(--color-5);
}

.time-input {
  width: 70px;
  font-size: 1.5rem;
  text-align: center;
  padding: 5px;
  background: var(--color-1);
  border: 4px solid var(--color-5);
}

.time-input::-webkit-outer-spin-button,
.time-input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}
.time-input[type=number] {
  -moz-appearance: textfield;
}

.set-custom-btn {
  background-color: var(--color-5);
  color: var(--color-1);
  padding: 8px 15px;
  font-size: 1.1rem;
  margin-left: 10px;
}
.set-custom-btn:hover { background-color: var(--color-4); color: var(--color-5); }

/* === 控制按鈕區 === */
.controls-row {
  display: flex;
  gap: 20px;
  width: 100%;
}

.start-btn {
  flex: 2;
  background-color: #10b981;
  color: white;
  font-size: 1.8rem;
  padding: 15px;
}
.start-btn:disabled { background-color: #9ca3af; cursor: not-allowed; transform: none; box-shadow: none; }

.pause-btn {
  flex: 2;
  background-color: var(--color-4);
  font-size: 1.8rem;
  padding: 15px;
}

.reset-btn {
  flex: 1;
  font-size: 1.5rem;
}

/* === 動畫效果 === */
@keyframes blink {
  50% { opacity: 0.5; }
}

.bounce-enter-active {
  animation: bounce-in 0.5s;
}
@keyframes bounce-in {
  0% { transform: translateX(-50%) scale(0); }
  50% { transform: translateX(-50%) scale(1.2); }
  100% { transform: translateX(-50%) scale(1); }
}

@media (max-width: 600px) {
  .timer-digits { font-size: 5rem; }
  .controls-row { flex-direction: column; }
  .custom-time-row { flex-wrap: wrap; }
}
</style>