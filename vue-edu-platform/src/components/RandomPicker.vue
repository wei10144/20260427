<template>
  <div class="picker-dashboard">
    <div class="card display-card">
      <h3 class="display-title" :class="{ 'is-visible': hasDrawn }">🎉 恭喜以下得獎者 🎉</h3>
      
      <div class="slot-machine-wrapper">
        <div class="slot-machine">
          <div class="slot-target-line"></div>
          <div class="slot-track" :style="trackStyle">
            <div v-for="(item, index) in slotItems" :key="index" class="slot-item">
              {{ item }}
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="action-row">
      <button @click="pickRandom" :disabled="isRolling" class="mega-spin-btn">
        ▶ {{ isRolling ? '轉動中...' : '單次抽籤' }}
      </button>
      <button class="reset-icon-btn" @click="resetList" title="重置名單">↺</button>
    </div>

    <div class="scroll-hint">
      向下滑動修改設定<br>▼
    </div>

    <div class="card settings-card">
      <h3 class="settings-title">⚙️ 抽籤設定</h3>
      
      <div class="settings-layout">
        <div class="list-input-area">
          <label>名單輸入 (每行一個)</label>
          <textarea v-model="listInput" placeholder="請輸入名單..." rows="6"></textarea>
        </div>
        
        <div class="options-area">
          <label class="checkbox-label">
            <input type="checkbox" v-model="excludeSelected"> 
            <span class="custom-checkbox"></span>
            抽中後自動從名單排除
          </label>
          <p class="status-text">目前名單剩餘人數：{{ remainingCount }} 人</p>
        </div>
      </div>
    </div>

    <audio id="drum-sound" src="/drum-roll-gaming-sound-effect-hd.mp3" preload="auto"></audio>
    <audio id="cheer-sound" src="https://actions.google.com/sounds/v1/crowds/crowd_loud_cheer.ogg" preload="auto"></audio>
  </div>
</template>

<script setup>
import { ref, nextTick, computed } from 'vue'

const listInput = ref('第一組\n第二組\n第三組\n第四組\n第五組\n第六組')
const excludeSelected = ref(true)
const isRolling = ref(false)
const hasDrawn = ref(false)

const slotItems = ref(['等待抽籤...'])
const trackStyle = ref({ transform: 'translateY(0px)', transition: 'none' })
const itemHeight = 120 

const remainingCount = computed(() => {
  return listInput.value.split('\n').filter(item => item.trim() !== '').length
})

const resetList = () => {
  if(confirm('確定要將名單恢復為預設的六組嗎？')) {
    listInput.value = '第一組\n第二組\n第三組\n第四組\n第五組\n第六組'
    slotItems.value = ['等待抽籤...']
    trackStyle.value = { transform: 'translateY(0px)', transition: 'none' }
    hasDrawn.value = false 
  }
}

// 🎆 觸發彩炮的專屬函數 (動態載入 canvas-confetti)
const fireConfetti = () => {
  const launch = () => {
    window.confetti({
      particleCount: 150, // 噴發 150 片彩紙
      spread: 100,        // 噴發角度範圍
      origin: { y: 0.6 }, // 噴發點 (稍微偏畫面下方往上噴)
      colors: ['#ef4444', '#10b981', '#3b82f6', '#f59e0b', '#8b5cf6', '#ffffff'] // 配合你的像素 UI 色系
    })
  }

  // 檢查是否已經載入過套件，沒有的話就動態載入
  if (window.confetti) {
    launch()
  } else {
    const script = document.createElement('script')
    script.src = 'https://cdn.jsdelivr.net/npm/canvas-confetti@1.6.0/dist/confetti.browser.min.js'
    script.onload = launch
    document.head.appendChild(script)
  }
}

const pickRandom = async () => {
  let items = listInput.value.split('\n').filter(item => item.trim() !== '')
  if (items.length === 0) {
    alert('名單已經空囉！請重新輸入。')
    return
  }

  isRolling.value = true
  hasDrawn.value = false 

  // 🔊 1. 開始抽籤，播放擊鼓聲
  const drumSfx = document.getElementById('drum-sound')
  if (drumSfx) {
    drumSfx.currentTime = 0 
    drumSfx.play().catch(err => console.log('音效被阻擋:', err))
  }

  const winnerIndex = Math.floor(Math.random() * items.length)
  const finalWinner = items[winnerIndex]

  const track = []
  for(let i = 0; i < 20; i++) {
    track.push(items[Math.floor(Math.random() * items.length)])
  }
  track.push(finalWinner)
  
  slotItems.value = track
  trackStyle.value = { transform: 'translateY(0px)', transition: 'none' }

  await nextTick()

  // 啟動 3.5 秒視覺滾動動畫
  setTimeout(() => {
    const targetY = -((track.length - 1) * itemHeight)
    trackStyle.value = {
      transform: `translateY(${targetY}px)`,
      transition: 'transform 3.5s cubic-bezier(0.15, 0.85, 0.15, 1)'
    }
  }, 50)

  // 3.6 秒後動畫停止
  setTimeout(() => {
    isRolling.value = false
    hasDrawn.value = true 
    
    // 🔊 2. 暫停擊鼓聲，播放歡呼聲
    if (drumSfx) drumSfx.pause()
    const cheerSfx = document.getElementById('cheer-sound')
    if (cheerSfx) {
      cheerSfx.currentTime = 0
      cheerSfx.play().catch(err => console.log('音效被阻擋:', err))
    }

    // 🎆 3. 噴發彩炮動畫！
    fireConfetti()

    if (excludeSelected.value) {
      listInput.value = items.filter(item => item !== finalWinner).join('\n')
    }
    
    slotItems.value = [finalWinner]
    trackStyle.value = { transform: 'translateY(0px)', transition: 'none' }
  }, 3600)
}
</script>

<style scoped>
.picker-dashboard {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
}

/* === 上方：展示區 === */
.display-card {
  width: 100%;
  max-width: 800px;
  text-align: center;
  padding: 40px 20px;
  background-color: var(--color-2);
  margin-bottom: 20px;
  border: 4px solid var(--color-5);
  box-shadow: 6px 6px 0px var(--color-4);
  position: relative; 
  z-index: 10;
}

.display-title {
  color: var(--color-5);
  font-size: 1.5rem;
  margin-bottom: 30px;
  display: inline-block;
  border: 2px dashed var(--color-5);
  padding: 10px 30px;
  background: var(--color-1);
  /* 預設隱藏並加入彈出動畫 */
  opacity: 0; 
  transition: opacity 0.5s ease, transform 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  transform: scale(0.8);
}

.display-title.is-visible {
  opacity: 1;
  transform: scale(1);
}

.slot-machine-wrapper {
  display: flex;
  justify-content: center;
}

.slot-machine {
  width: 100%;
  max-width: 400px;
  height: 120px;
  background: var(--color-1);
  border: 6px solid var(--color-5);
  box-shadow: inset 5px 5px 15px rgba(0,0,0,0.5), 8px 8px 0px var(--color-4);
  overflow: hidden;
  position: relative;
}

.slot-machine::before, .slot-machine::after {
  content: ''; position: absolute; top: 50%; transform: translateY(-50%);
  border-top: 15px solid transparent; border-bottom: 15px solid transparent; z-index: 10;
}
.slot-machine::before { left: 0; border-left: 20px solid #ef4444; }
.slot-machine::after { right: 0; border-right: 20px solid #ef4444; }

.slot-target-line {
  position: absolute; top: 0; left: 0; width: 100%; height: 120px;
  background: rgba(255, 255, 255, 0.1); border-top: 2px dashed rgba(127, 85, 57, 0.3);
  border-bottom: 2px dashed rgba(127, 85, 57, 0.3); pointer-events: none; z-index: 5;
}

.slot-track { width: 100%; display: flex; flex-direction: column; }
.slot-item {
  height: 120px; display: flex; align-items: center; justify-content: center;
  font-size: 4rem; color: var(--color-5); font-weight: bold;
  text-shadow: 3px 3px 0px var(--color-3);
}

/* === 中央：操作按鈕區 === */
.action-row {
  display: flex;
  gap: 15px;
  width: 100%;
  max-width: 800px;
  margin-bottom: 30px;
  position: relative;
  z-index: 10;
}

.mega-spin-btn {
  flex: 1;
  font-size: 2rem;
  padding: 20px;
  background-color: var(--color-4);
  color: var(--color-1);
  box-shadow: 6px 6px 0px var(--color-5);
  border: 4px solid var(--color-5);
}

.mega-spin-btn:hover:not(:disabled) {
  background-color: var(--color-3);
  color: var(--color-5);
}

.reset-icon-btn {
  width: 80px;
  font-size: 2.5rem;
  background-color: var(--color-2);
  color: var(--color-5);
  box-shadow: 6px 6px 0px var(--color-5);
  border: 4px solid var(--color-5);
}

/* === 向下提示 === */
.scroll-hint {
  text-align: center;
  color: var(--color-5);
  opacity: 0.6;
  margin-bottom: 30px;
  line-height: 1.5;
  animation: floatDown 2s infinite;
}

@keyframes floatDown {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(10px); }
}

/* === 下方：設定區 === */
.settings-card {
  width: 100%;
  max-width: 800px;
  background-color: var(--color-2);
  border: 4px solid var(--color-5);
  padding: 20px;
  box-shadow: 6px 6px 0px var(--color-4);
  position: relative;
  z-index: 10;
}

.settings-title {
  margin-top: 0;
  border-bottom: 4px dashed var(--color-4);
  padding-bottom: 10px;
}

.settings-layout {
  display: flex;
  gap: 30px;
  margin-top: 20px;
}

@media (max-width: 600px) {
  .settings-layout { flex-direction: column; }
}

.list-input-area {
  flex: 2;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.list-input-area label { font-weight: bold; }
.list-input-area textarea { 
  resize: vertical; 
  font-size: 1.2rem; 
  padding: 10px;
  background: var(--color-1);
  border: 4px solid var(--color-5);
}

.options-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 20px;
  background: var(--color-1);
  padding: 20px;
  border: 4px solid var(--color-5);
}

.checkbox-label {
  display: flex; align-items: center; gap: 10px; cursor: pointer; font-size: 1.1rem; font-weight: bold;
}
.checkbox-label input { display: none; }
.custom-checkbox {
  width: 24px; height: 24px; background: var(--color-1); border: 3px solid var(--color-5); display: inline-block; position: relative;
}
.checkbox-label input:checked + .custom-checkbox::after {
  content: '✔'; position: absolute; top: -5px; left: 2px; color: #10b981; font-size: 1.5rem;
}

.status-text {
  font-size: 1.2rem;
  color: #ef4444;
  font-weight: bold;
  text-shadow: 1px 1px 0px var(--color-3);
  margin: 0;
}
</style>