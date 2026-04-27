<template>
  <div class="card scoreboard-container">
    <h2>🏆 小組競賽記分板</h2>

    <div class="control-panel">
      <div class="group-controls">
        <button @click="addGroup">➕ 新增小組</button>
        <button class="outline" @click="removeLastGroup" :disabled="groups.length === 0">➖ 減少小組</button>
      </div>
      <button class="outline small-btn reset-btn" @click="resetConfirm">🗑️ 重設所有數據</button>
    </div>

    <transition-group name="list" tag="div" class="scoreboard">
      <div 
        v-for="(group) in rankedGroups" 
        :key="group.id" 
        class="score-card"
        :class="{ 'top-three': group.rank <= 3 }"
      >
        <button class="delete-btn" @click="removeGroup(group.id)" title="刪除此小組">✖</button>

        <div class="rank-badge">
          <span v-if="group.rank === 1" class="medal">🥇</span>
          <span v-else-if="group.rank === 2" class="medal">🥈</span>
          <span v-else-if="group.rank === 3" class="medal">🥉</span>
          第 {{ group.rank }} 名
        </div>

        <input 
          class="group-name-input" 
          type="text" 
          :value="group.name" 
          @input="updateName(group.id, $event.target.value)"
          placeholder="請輸入小組名稱"
        >

        <div class="score-display">{{ group.score }}</div>

        <div class="score-actions">
          <button class="outline" @click="updateScore(group.id, -1)">-1</button>
          <button @click="updateScore(group.id, 1)">+1</button>
          <button @click="updateScore(group.id, 5)">+5</button>
        </div>
      </div>
    </transition-group>
    
    <div v-if="groups.length === 0" class="empty-state">
      目前沒有任何小組，請點擊上方「➕ 新增小組」開始競賽！
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'

const groups = ref([])

// 🎵 產生 8-bit 復古遊戲音效的核心函數
const playRetroSound = (type) => {
  // 建立音訊上下文
  const audioCtx = new (window.AudioContext || window.webkitAudioContext)()
  const osc = audioCtx.createOscillator()
  const gain = audioCtx.createGain()

  osc.connect(gain)
  gain.connect(audioCtx.destination)

  if (type === 'up') {
    // 🪙 加分：短促高頻的吃金幣聲 (Square wave)
    osc.type = 'square'
    osc.frequency.setValueAtTime(800, audioCtx.currentTime)
    osc.frequency.setValueAtTime(1200, audioCtx.currentTime + 0.05) // 音調瞬間往上揚
    
    gain.gain.setValueAtTime(0.1, audioCtx.currentTime) // 控制音量
    gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.15) // 短促淡出
    
    osc.start(audioCtx.currentTime)
    osc.stop(audioCtx.currentTime + 0.15)
  } else if (type === 'down') {
    // 💥 扣分：低頻下降的倒楣/受傷聲 (Sawtooth wave)
    osc.type = 'sawtooth'
    osc.frequency.setValueAtTime(300, audioCtx.currentTime)
    osc.frequency.exponentialRampToValueAtTime(50, audioCtx.currentTime + 0.25) // 音調往下沉
    
    gain.gain.setValueAtTime(0.1, audioCtx.currentTime)
    gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.25)
    
    osc.start(audioCtx.currentTime)
    osc.stop(audioCtx.currentTime + 0.25)
  }
}

// 從 LocalStorage 載入資料，若無則預設產生 4 組
onMounted(() => {
  const savedData = localStorage.getItem('pixel_scoreboard_data')
  if (savedData) {
    groups.value = JSON.parse(savedData)
  } else {
    for (let i = 1; i <= 4; i++) {
      groups.value.push({ id: Date.now() + i, name: `第 ${i} 組`, score: 0 })
    }
  }
})

// 監聽資料變動並自動儲存
watch(groups, (newVal) => {
  localStorage.setItem('pixel_scoreboard_data', JSON.stringify(newVal))
}, { deep: true })

const addGroup = () => {
  groups.value.push({
    id: Date.now(),
    name: `第 ${groups.value.length + 1} 組`,
    score: 0
  })
}

const removeLastGroup = () => {
  if (groups.value.length > 0) {
    groups.value.pop()
  }
}

const removeGroup = (id) => {
  groups.value = groups.value.filter(g => g.id !== id)
}

const updateName = (id, newName) => {
  const group = groups.value.find(g => g.id === id)
  if (group) group.name = newName
}

const rankedGroups = computed(() => {
  const sorted = [...groups.value].sort((a, b) => b.score - a.score)
  return sorted.map((group, index, array) => {
    let rank = index + 1
    if (index > 0 && group.score === array[index - 1].score) {
      rank = array[index - 1].rank 
    }
    return { ...group, rank }
  })
})

const updateScore = (groupId, points) => {
  const group = groups.value.find(g => g.id === groupId)
  if (group) {
    group.score += points
    
    // 根據加減分呼叫對應的 8-bit 音效
    if (points > 0) {
      playRetroSound('up')
    } else if (points < 0) {
      playRetroSound('down')
    }
  }
}

const resetConfirm = () => {
  if (confirm('確定要清除所有計分與小組嗎？此動作無法復原。')) {
    localStorage.removeItem('pixel_scoreboard_data')
    groups.value = []
  }
}
</script>

<style scoped>
.scoreboard-container {
  min-height: 400px;
}

/* 控制面板樣式 */
.control-panel {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 15px;
  background: var(--color-2);
  padding: 15px;
  border: 4px solid var(--color-5);
  margin-bottom: 25px;
}

.group-controls {
  display: flex;
  gap: 10px;
}

.small-btn {
  font-size: 0.9rem;
  padding: 8px 12px;
}

.reset-btn {
  background-color: #ef4444 !important;
  color: white !important;
  border-color: #b91c1c !important;
}

/* 計分板佈局 */
.scoreboard {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 20px;
}

.score-card {
  border: 4px solid var(--color-5);
  background: var(--color-1);
  padding: 20px 15px 15px 15px;
  text-align: center;
  position: relative;
  transition: all 0.5s cubic-bezier(0.15, 0.85, 0.15, 1);
}

.score-card.top-three {
  background: var(--color-2);
  border-width: 6px;
}

/* 刪除個別小組按鈕 */
.delete-btn {
  position: absolute;
  top: 5px;
  right: 5px;
  background: transparent;
  color: var(--color-5);
  border: none;
  box-shadow: none;
  font-size: 1.2rem;
  padding: 0;
  margin: 0;
  opacity: 0.5;
  transition: opacity 0.2s;
}

.delete-btn:hover {
  background: transparent;
  color: #ef4444;
  opacity: 1;
  box-shadow: none;
  transform: scale(1.2);
}

.rank-badge {
  background: var(--color-5);
  color: var(--color-1);
  display: inline-block;
  padding: 4px 12px;
  font-size: 0.9rem;
  margin-bottom: 10px;
}

.medal {
  font-size: 1.4rem;
  margin-right: 5px;
  vertical-align: middle;
}

/* 可編輯的小組名稱輸入框 */
.group-name-input {
  width: 90%;
  text-align: center;
  font-size: 1.4rem;
  font-weight: bold;
  background: transparent;
  border: 2px dashed transparent;
  color: var(--color-5);
  margin: 5px auto;
  padding: 5px;
  transition: all 0.2s;
  display: block;
}

.group-name-input:hover, .group-name-input:focus {
  border-color: var(--color-4);
  background: rgba(255, 255, 255, 0.5);
  outline: none;
  box-shadow: none;
}

.score-display {
  font-size: 3.5rem;
  color: var(--color-5);
  text-shadow: 3px 3px 0px var(--color-3);
  margin: 10px 0;
}

.score-actions {
  display: flex;
  justify-content: center;
  gap: 8px;
}

.empty-state {
  text-align: center;
  padding: 40px;
  color: var(--color-5);
  font-size: 1.2rem;
  opacity: 0.7;
}

/* 位移動畫 */
.list-move {
  transition: transform 0.6s ease;
}
.list-enter-active, .list-leave-active {
  transition: all 0.5s ease;
}
.list-enter-from, .list-leave-to {
  opacity: 0;
  transform: scale(0.8);
}
</style>    