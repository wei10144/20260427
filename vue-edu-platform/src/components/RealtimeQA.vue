<template>
  <div class="qa-container">
    
    <div v-if="role === null" class="role-selection card">
      <h2>📡 請選擇您的身分</h2>
      <div class="role-buttons">
        <button class="role-btn teacher-btn" @click="selectRole('teacher')">
          <span class="icon">👨‍🏫</span> 建立互動教室<br><small>(教師端)</small>
        </button>
        <button class="role-btn student-btn" @click="selectRole('student')">
          <span class="icon">🙋‍♂️</span> 加入互動課堂<br><small>(學生端)</small>
        </button>
      </div>
    </div>

    <div v-else-if="role === 'teacher'" class="teacher-dashboard">
      <div class="left-panel">
        <div class="card info-card">
          <div class="card-header">
            <h3>((•)) 教室資訊</h3>
            <span class="student-count">人數: {{ roomState.students.length }} 🟢</span>
          </div>
          <div class="code-display">
            <p>請學生輸入代碼或掃描加入</p>
            <div class="room-code">{{ roomState.code }}</div>
            
            <button class="qr-toggle-btn" @click="showQR = !showQR">
              {{ showQR ? '隱藏 QR Code' : '📱 顯示 QR Code' }}
            </button>
            
            <div v-if="showQR" class="qr-code-wrapper">
              <img :src="qrCodeUrl" alt="Room QR Code" class="pixel-qr" />
            </div>
          </div>
        </div>

        <div class="card interaction-card">
          <div class="card-header">
            <h3>🚀 課堂互動</h3>
          </div>
          <div class="status-box" :class="{ active: roomState.isActive }">
            <div class="status-icon">{{ roomState.isActive ? '((•))' : '⏸' }}</div>
            <h4>{{ roomState.isActive ? '作答收集中' : '等待發題' }}</h4>
            <p>{{ getQuestionTypeName() }}</p>
          </div>
          
          <div v-if="!roomState.isActive" class="control-buttons">
            <button @click="startQuestion('mcq')">發起單選題 (A,B,C,D)</button>
            <button @click="startQuestion('open')">發起開放式問答</button>
          </div>
          <div v-else class="control-buttons">
            <button class="stop-btn" @click="closeQuestion">⏹ 停止作答並結算</button>
          </div>
        </div>
      </div>

      <div class="right-panel card">
        <div class="card-header">
          <h3>📊 作答統計</h3>
          <span>作答進度: {{ roomState.results.length }} / {{ roomState.students.length }}</span>
        </div>
        
        <div v-if="roomState.type === 'mcq'" class="chart-container">
          <div class="bar-chart">
            <div class="bar-group" v-for="opt in ['A', 'B', 'C', 'D']" :key="opt">
              <div class="bar-value">{{ getMcqCount(opt) }}</div>
              <div class="bar-track">
                <div class="bar-fill" :style="{ height: getMcqPercentage(opt) + '%' }"></div>
              </div>
              <div class="bar-label">{{ opt }}</div>
            </div>
          </div>
        </div>

        <div v-else-if="roomState.type === 'open'" class="open-answers-container">
          <div v-if="roomState.results.length === 0" class="empty-hint">等待學生提交答案...</div>
          <div v-for="(res, index) in roomState.results" :key="index" class="open-reply">
            <span class="reply-author">{{ res.nickname }}：</span>
            <span class="reply-content">{{ res.answer }}</span>
          </div>
        </div>

        <div v-else class="empty-chart">
          請從左側面板發起互動問答
        </div>

        <div class="student-list-bar">
          <h4>連線名單 & 狀態：</h4>
          <div class="student-tags">
            <span 
              v-for="student in roomState.students" 
              :key="student.nickname" 
              class="student-tag"
              :class="{ answered: hasStudentAnswered(student.nickname) }"
            >
              {{ student.nickname }}
            </span>
          </div>
        </div>
      </div>
    </div>

    <div v-else-if="role === 'student'" class="student-dashboard">
      <div class="header-bar">
        <h2>🙋‍♂️ 學生作答區</h2>
        <button class="outline small-btn" @click="role = null">離開</button>
      </div>

      <div v-if="!joined" class="join-room card">
        <h2>加入互動課堂</h2>
        <p class="subtitle">請確認教室代碼，並輸入您的暱稱</p>
        
        <div class="input-group">
          <label>教室代碼 (Room ID)</label>
          <input type="text" v-model="joinCode" placeholder="例如：8273" maxlength="4">
        </div>

        <div class="input-group">
          <label>你的名字 (暱稱) <span class="required">*必填</span></label>
          <input type="text" v-model="nickname" placeholder="請輸入姓名或座號" autofocus>
        </div>

        <button class="mega-btn" @click="joinRoom" :disabled="!joinCode || !nickname">進入教室</button>
        <p v-if="joinError" class="error-text">❌ 找不到該代碼，請確認後重試。</p>
      </div>

      <div v-else class="answer-section card">
        <div class="status-badge">✅ 已連線：{{ joinCode }} | 👤 {{ nickname }}</div>
        
        <div v-if="!roomState.isActive" class="waiting-state">
          <div class="spinner">⏳</div>
          <h3>等待老師發起下一道題目...</h3>
          <p>請留意前方大螢幕的指示</p>
        </div>

        <div v-else-if="myAnswerSubmitted" class="waiting-state success">
          <div class="spinner">🎉</div>
          <h3>已成功送出答案！</h3>
          <p>請看大螢幕等待老師公布結果</p>
        </div>

        <div v-else class="active-question">
          <h3 class="q-title">請選擇或輸入您的答案：</h3>
          
          <div v-if="roomState.type === 'mcq'" class="mcq-grid">
            <button class="mcq-btn" @click="submitAnswer('A')">A</button>
            <button class="mcq-btn" @click="submitAnswer('B')">B</button>
            <button class="mcq-btn" @click="submitAnswer('C')">C</button>
            <button class="mcq-btn" @click="submitAnswer('D')">D</button>
          </div>

          <div v-if="roomState.type === 'open'" class="open-input">
            <textarea v-model="openAnswer" rows="5" placeholder="請在此輸入您的想法..."></textarea>
            <button class="mega-btn" @click="submitAnswer(openAnswer)" :disabled="!openAnswer.trim()">送出答案</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'

const role = ref(null) 
let syncInterval = null

// 房間資料結構
const roomState = ref({
  code: '',
  isActive: false,
  type: '', 
  students: [], 
  results: []   
})

// === QR Code 邏輯 ===
const showQR = ref(false)

// 動態產生包含房間代碼的專屬網址 QR Code
const qrCodeUrl = computed(() => {
  if (!roomState.value.code) return ''
  // 組合當前網址，並加上 ?room=代碼
  const url = `${window.location.origin}${window.location.pathname}?room=${roomState.value.code}`
  // 使用開源 API 產生圖片
  return `https://api.qrserver.com/v1/create-qr-code/?size=200x200&data=${encodeURIComponent(url)}`
})

// === 共用資料同步邏輯 ===
const selectRole = (selectedRole) => {
  role.value = selectedRole
  if (selectedRole === 'teacher') {
    const newCode = Math.floor(1000 + Math.random() * 9000).toString()
    roomState.value = { code: newCode, isActive: false, type: '', students: [], results: [] }
    saveToStorage()
  }
  syncInterval = setInterval(loadFromStorage, 1000)
}

const loadFromStorage = () => {
  const data = localStorage.getItem('pixel_qa_room')
  if (data) {
    const parsed = JSON.parse(data)
    roomState.value = parsed
    
    // 如果是學生端，檢查老師是否關閉了題目，以重置作答狀態
    if (role.value === 'student' && !parsed.isActive) {
      myAnswerSubmitted.value = false
    }
  }
}

const saveToStorage = () => {
  localStorage.setItem('pixel_qa_room', JSON.stringify(roomState.value))
}

onUnmounted(() => {
  if (syncInterval) clearInterval(syncInterval)
})

// === 教師端邏輯 ===
const startQuestion = (type) => {
  roomState.value.isActive = true
  roomState.value.type = type
  roomState.value.results = [] 
  saveToStorage()
}

const closeQuestion = () => {
  roomState.value.isActive = false
  saveToStorage()
}

const getQuestionTypeName = () => {
  if (!roomState.value.isActive) return '---'
  return roomState.value.type === 'mcq' ? '單選 (A, B, C, D)' : '開放式問答'
}

const getMcqCount = (opt) => {
  return roomState.value.results.filter(r => r.answer === opt).length
}

const getMcqPercentage = (opt) => {
  const total = roomState.value.results.length
  if (total === 0) return 0
  return Math.round((getMcqCount(opt) / total) * 100)
}

const hasStudentAnswered = (nickname) => {
  return roomState.value.results.some(r => r.nickname === nickname)
}

// === 學生端邏輯 ===
const joined = ref(false)
const joinCode = ref('')
const nickname = ref('')
const joinError = ref(false)
const myAnswerSubmitted = ref(false)
const openAnswer = ref('')

// 🎯 初始化：檢查網址是否有代碼 (處理掃描 QR Code 進來的學生)
onMounted(() => {
  const urlParams = new URLSearchParams(window.location.search)
  const roomParam = urlParams.get('room')
  if (roomParam) {
    joinCode.value = roomParam   // 自動填入教室代碼
    role.value = 'student'       // 自動切換為學生端畫面
    // 啟動資料同步
    syncInterval = setInterval(loadFromStorage, 1000)
  }
})

const joinRoom = () => {
  loadFromStorage()
  if (roomState.value.code === joinCode.value) {
    joined.value = true
    joinError.value = false
    // 將學生加入名單
    if (!roomState.value.students.some(s => s.nickname === nickname.value)) {
      roomState.value.students.push({ nickname: nickname.value })
      saveToStorage()
    }
  } else {
    joinError.value = true
  }
}

const submitAnswer = (answer) => {
  if (!answer) return
  loadFromStorage() 
  roomState.value.results.push({ nickname: nickname.value, answer })
  saveToStorage()
  
  myAnswerSubmitted.value = true
  openAnswer.value = '' 
}
</script>

<style scoped>
.qa-container { width: 100%; max-width: 1000px; margin: 0 auto; }
.card { background: var(--color-2); border: 4px solid var(--color-5); padding: 20px; box-shadow: 6px 6px 0px var(--color-4); margin-bottom: 20px; }
.card-header { display: flex; justify-content: space-between; align-items: center; border-bottom: 3px dashed var(--color-5); padding-bottom: 10px; margin-bottom: 15px; }
.card-header h3 { margin: 0; }
.small-btn { padding: 5px 15px; font-size: 0.9rem; }
.mega-btn { width: 100%; padding: 15px; font-size: 1.5rem; background: var(--color-4); margin-top: 20px; }

/* === 身分選擇 === */
.role-selection { text-align: center; padding: 50px 20px; }
.role-buttons { display: flex; justify-content: center; flex-wrap: wrap; gap: 30px; margin-top: 30px; }
.role-btn { width: 260px; padding: 30px; font-size: 1.5rem; display: flex; flex-direction: column; align-items: center; gap: 10px; }
.role-btn .icon { font-size: 3rem; }
.student-btn { background: #3b82f6; color: white; border-color: #1e3a8a; box-shadow: 6px 6px 0px #1e3a8a; }

/* === 教師端雙欄排版 === */
.teacher-dashboard { display: grid; grid-template-columns: 320px 1fr; gap: 20px; align-items: start; }
@media (max-width: 800px) { .teacher-dashboard { grid-template-columns: 1fr; } }

/* 左側：教室資訊與互動 */
.student-count { font-weight: bold; color: #10b981; }
.code-display { text-align: center; background: var(--color-1); border: 4px solid var(--color-5); padding: 20px; margin-bottom: 15px; }
.code-display p { margin: 0 0 10px 0; font-size: 0.9rem; }
.room-code { font-size: 3.5rem; font-weight: bold; letter-spacing: 5px; text-shadow: 2px 2px 0px var(--color-3); }

/* QR Code 樣式 */
.qr-toggle-btn {
  margin-top: 15px;
  background: transparent;
  border: 2px solid var(--color-5);
  color: var(--color-5);
  padding: 8px 15px;
  font-size: 1rem;
  cursor: pointer;
  width: 100%;
}
.qr-toggle-btn:hover { background: var(--color-4); color: var(--color-1); }
.qr-code-wrapper {
  margin-top: 15px;
  background: white;
  padding: 10px;
  display: inline-block;
  border: 4px solid var(--color-5);
}
.pixel-qr {
  width: 150px;
  height: 150px;
  image-rendering: pixelated; /* 強化像素風格 */
}

.status-box { background: var(--color-1); border: 4px solid var(--color-5); text-align: center; padding: 20px; margin-bottom: 15px; transition: all 0.3s; }
.status-box.active { border-color: #10b981; box-shadow: inset 0 0 10px rgba(16, 185, 129, 0.2); }
.status-icon { font-size: 2.5rem; color: #10b981; margin-bottom: 10px; }
.status-box.active .status-icon { animation: blink 1.5s infinite; }
.control-buttons { display: flex; flex-direction: column; gap: 10px; }
.control-buttons button { padding: 12px; font-size: 1.1rem; }
.stop-btn { background-color: #ef4444; color: white; border-color: #b91c1c; box-shadow: 4px 4px 0px #b91c1c; }

/* 右側：統計圖表 */
.right-panel { display: flex; flex-direction: column; min-height: 500px; }
.chart-container { flex: 1; display: flex; justify-content: center; align-items: flex-end; padding: 40px 20px; background: var(--color-1); border: 4px solid var(--color-5); margin-bottom: 20px; }
.bar-chart { display: flex; justify-content: space-around; width: 100%; max-width: 500px; height: 250px; }
.bar-group { display: flex; flex-direction: column; align-items: center; justify-content: flex-end; width: 50px; }
.bar-value { font-weight: bold; margin-bottom: 5px; font-size: 1.2rem; }
.bar-track { width: 100%; height: 200px; background: var(--color-2); border: 3px solid var(--color-5); display: flex; align-items: flex-end; position: relative; }
.bar-fill { width: 100%; background: var(--color-4); transition: height 0.5s cubic-bezier(0.15, 0.85, 0.15, 1); }
.bar-label { margin-top: 10px; font-size: 1.5rem; font-weight: bold; }

.open-answers-container { flex: 1; background: var(--color-1); border: 4px solid var(--color-5); padding: 15px; margin-bottom: 20px; max-height: 350px; overflow-y: auto; }
.open-reply { background: var(--color-2); border: 2px solid var(--color-5); padding: 10px 15px; margin-bottom: 10px; }
.reply-author { font-weight: bold; color: var(--color-5); }
.empty-chart, .empty-hint { flex: 1; display: flex; align-items: center; justify-content: center; color: var(--color-5); opacity: 0.6; font-size: 1.2rem; min-height: 200px; }

/* 底部連線名單 */
.student-list-bar { padding-top: 15px; border-top: 3px dashed var(--color-5); }
.student-list-bar h4 { margin: 0 0 10px 0; }
.student-tags { display: flex; flex-wrap: wrap; gap: 8px; }
.student-tag { background: var(--color-1); border: 2px solid var(--color-5); padding: 4px 10px; font-size: 0.9rem; }
.student-tag.answered { background: #10b981; color: white; border-color: #047857; }

/* === 學生端 === */
.header-bar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 4px solid var(--color-5); padding-bottom: 10px; }
.join-room { max-width: 500px; margin: 40px auto; text-align: center; }
.subtitle { opacity: 0.8; margin-bottom: 30px; }
.input-group { text-align: left; margin-bottom: 20px; }
.input-group label { display: block; font-weight: bold; margin-bottom: 8px; }
.required { color: #ef4444; font-size: 0.8rem; }
.input-group input { width: 100%; font-size: 1.5rem; padding: 10px; background: var(--color-1); border: 4px solid var(--color-5); }
.error-text { color: #ef4444; font-weight: bold; margin-top: 15px; }

.status-badge { display: inline-block; background: var(--color-1); border: 2px solid var(--color-5); padding: 8px 15px; font-weight: bold; margin-bottom: 30px; }
.waiting-state { text-align: center; padding: 50px 20px; }
.waiting-state.success { color: #10b981; }
.spinner { font-size: 4rem; animation: bounce 1s infinite alternate; margin-bottom: 20px; }
.q-title { text-align: center; font-size: 1.5rem; margin-bottom: 30px; }
.mcq-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; max-width: 500px; margin: 0 auto; }
.mcq-btn { font-size: 3rem; padding: 40px 20px; background: var(--color-4); box-shadow: 6px 6px 0px var(--color-5); border: 4px solid var(--color-5); }
.open-input { max-width: 600px; margin: 0 auto; }
.open-input textarea { width: 100%; font-size: 1.2rem; padding: 15px; background: var(--color-1); border: 4px solid var(--color-5); }

@keyframes blink { 50% { opacity: 0.3; } }
@keyframes bounce { from { transform: translateY(0); } to { transform: translateY(-15px); } }
</style>