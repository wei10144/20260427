<template>
  <div class="exam-container">
    <div class="header">
      <h2>📜 歷史探險測驗</h2>
    </div>

    <div v-if="isSubmitted" class="card result-board">
      <h3>任務結算</h3>
      <div class="score-display">
        得分：<span :class="{ 'high-score': score >= 80, 'low-score': score < 60 }">{{ score }}</span> / 100
      </div>
      <p>答對題數：{{ correctCount }} / 5 題</p>
      <p class="hint-text">快往下捲動查看正確答案與歷史解析吧！</p>
      <button class="reset-btn" @click="resetExam">重新挑戰</button>
    </div>

    <div class="card">
      <div class="status-bar">
        <h3 class="section-title">任務列表</h3>
        <p class="status-text" v-if="!isSubmitted">已完成 {{ answeredCount }} / 5 題</p>
        <p class="status-text" v-else>作答已結束，每題配分 20 分。</p>
      </div>
      
      <div class="questions-grid">
        <div 
          v-for="(q, index) in questions" 
          :key="q.id" 
          class="question-card"
          :class="{ 'correct-card': isSubmitted && checkCorrect(index), 'wrong-card': isSubmitted && !checkCorrect(index) }"
        >
          <div class="question-header">
            <span class="type-badge" :class="q.type">{{ getTypeLabel(q.type) }}</span>
            <h4>
              <span v-if="isSubmitted" class="result-icon">
                {{ checkCorrect(index) ? '✅' : '❌' }}
              </span>
              {{ q.title }}
            </h4>
          </div>

          <div class="options">
            <button 
              v-for="opt in q.options" 
              :key="opt"
              class="option-btn"
              :disabled="isSubmitted"
              :class="{ 
                'selected': isSelected(index, opt, q.type),
                'show-correct': isSubmitted && isOptionCorrect(index, opt),
                'show-wrong': isSubmitted && isSelected(index, opt, q.type) && !isOptionCorrect(index, opt)
              }"
              @click="selectAnswer(index, opt, q.type)"
            >
              <span v-if="q.type === 'multiple'" class="multi-check">
                {{ isSelected(index, opt, q.type) ? '☑' : '☐' }}
              </span>
              {{ opt }}
            </button>
          </div>

          <transition name="slide-down">
            <div v-if="isSubmitted" class="explanation-box">
              <div class="exp-title">💡 歷史解析：</div>
              <p>{{ q.explanation }}</p>
            </div>
          </transition>
        </div>
      </div>
    </div>

    <div v-if="!isSubmitted" class="card review-section">
      <h3 class="section-title">作答卷軸回顧</h3>
      <div v-if="answeredCount === 0" class="empty-state">您尚未選擇任何答案。</div>
      <ul v-else class="review-list">
        <li v-for="(q, index) in questions" :key="index">
          <div v-if="hasAnswered(index, q.type)" class="review-item">
            <span class="review-q">第 {{ index + 1 }} 題：</span>
            <span class="review-a">{{ formatAnswer(userAnswers[index]) }}</span>
          </div>
        </li>
      </ul>
      <button 
        v-if="answeredCount === questions.length" 
        class="submit-btn" 
        @click="submitExam"
      >
        繳交任務卷軸
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, reactive, onMounted } from 'vue'

// 精簡為 5 題
const questions = ref([
  {
    id: 1, type: 'single',
    title: '荷治時期，荷蘭人主要在臺灣南部建立的政治與軍事中心是哪座城堡？',
    options: ['聖地牙哥城', '紅毛城', '熱蘭遮城', '普羅民遮城'],
    correct: '熱蘭遮城',
    explanation: '荷蘭人在大員（今臺南安平）建立「熱蘭遮城」作為政治軍事核心；而「普羅民遮城」（今赤崁樓）則是後來擴建的行政與農業中心。'
  },
  {
    id: 2, type: 'tf',
    title: '鄭成功驅逐荷蘭人後，為解決軍糧問題而在臺灣實施「屯田制」。',
    options: ['⭕ 是 (True)', '❌ 非 (False)'],
    correct: '⭕ 是 (True)',
    explanation: '正確。鄭成功為解決數萬大軍的糧食問題，實施「寓兵於農」的屯田制度，部隊平時種田、戰時打仗。'
  },
  {
    id: 3, type: 'multiple',
    title: '下列哪些抗日事件發生在臺灣日治時期？（請選擇所有正確項目）',
    options: ['霧社事件', '噍吧哖事件', '二二八事件', '苗栗事件'],
    correct: ['霧社事件', '噍吧哖事件', '苗栗事件'],
    explanation: '二二八事件發生於戰後中華民國統治時期（1947年），其餘皆為日治時期的武裝抗日事件。'
  },
  {
    id: 4, type: 'single',
    title: '「牡丹社事件」發生於哪一年，促使清廷開始重視臺灣的防務與建設？',
    options: ['1874年', '1884年', '1895年', '1930年'],
    correct: '1874年',
    explanation: '1874年爆發牡丹社事件，日本出兵臺灣。此後清廷派沈葆楨來臺加強防務，開始積極建設。'
  },
  {
    id: 5, type: 'multiple',
    title: '關於清領初期的「渡臺禁令」，包含以下哪些規定？（請選擇所有正確項目）',
    options: ['必須事先申請渡航照單', '禁止攜帶家眷入臺', '規定只能從鹿港登陸', '嚴格限制粵地客家人渡臺'],
    correct: ['必須事先申請渡航照單', '禁止攜帶家眷入臺', '嚴格限制粵地客家人渡臺'],
    explanation: '渡臺禁令包含「照單渡臺、禁攜家眷、禁粵民渡臺」三大要點。初期只有開放臺南「鹿耳門」作為唯一合法登陸港口。'
  }
])

const userAnswers = reactive({})
const isSubmitted = ref(false)

const initAnswers = () => {
  questions.value.forEach((q, index) => {
    userAnswers[index] = q.type === 'multiple' ? [] : null
  })
}

onMounted(() => {
  initAnswers()
})

const getTypeLabel = (type) => {
  const labels = { 'single': '單選', 'tf': '是非', 'multiple': '複選' }
  return labels[type] || '未知'
}

const hasAnswered = (index, type) => {
  const ans = userAnswers[index]
  if (type === 'multiple') return Array.isArray(ans) && ans.length > 0
  return ans !== undefined && ans !== null
}

const answeredCount = computed(() => {
  return questions.value.filter((q, index) => hasAnswered(index, q.type)).length
})

const isSelected = (qIndex, option, type) => {
  if (type === 'multiple') return userAnswers[qIndex]?.includes(option)
  return userAnswers[qIndex] === option
}

const isOptionCorrect = (qIndex, option) => {
  const correctAns = questions.value[qIndex].correct
  if (Array.isArray(correctAns)) return correctAns.includes(option)
  return correctAns === option
}

// 判斷整題是否答對
const checkCorrect = (qIndex) => {
  const uAns = userAnswers[qIndex]
  const cAns = questions.value[qIndex].correct
  
  if (Array.isArray(cAns)) {
    if (!Array.isArray(uAns) || uAns.length !== cAns.length) return false
    return cAns.every(val => uAns.includes(val))
  }
  return uAns === cAns
}

// 計算答對題數
const correctCount = computed(() => {
  let count = 0
  questions.value.forEach((_, index) => {
    if (checkCorrect(index)) count++
  })
  return count
})

// 計算總分：每題 20 分
const score = computed(() => {
  return correctCount.value * 20
})

const selectAnswer = (qIndex, option, type) => {
  if (isSubmitted.value) return 
  
  if (type === 'single' || type === 'tf') {
    userAnswers[qIndex] = option
  } else if (type === 'multiple') {
    const ansArray = userAnswers[qIndex]
    const optIndex = ansArray.indexOf(option)
    if (optIndex > -1) ansArray.splice(optIndex, 1)
    else ansArray.push(option)
  }
}

const formatAnswer = (ans) => {
  return Array.isArray(ans) ? ans.join('、') : ans
}

const submitExam = () => {
  isSubmitted.value = true
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

const resetExam = () => {
  if (confirm('確定要重新測驗嗎？這將會清除目前的作答紀錄。')) {
    isSubmitted.value = false
    initAnswers()
    window.scrollTo({ top: 0, behavior: 'smooth' })
  }
}
</script>

<style scoped>
.exam-container { max-width: 900px; margin: 0 auto; }
.header { text-align: center; margin-bottom: 20px; }

.result-board {
  text-align: center;
  background-color: var(--color-2);
  border: 6px solid var(--color-5);
}
.score-display {
  font-size: 2rem;
  font-weight: bold;
  margin: 15px 0;
}
.high-score { color: #10b981; font-size: 3.5rem; text-shadow: 2px 2px 0px #047857; }
.low-score { color: #ef4444; font-size: 3.5rem; text-shadow: 2px 2px 0px #b91c1c; }
.hint-text { margin-bottom: 20px; color: var(--color-5); }

.status-bar {
  display: flex; justify-content: space-between; align-items: center;
  border-bottom: 4px dashed var(--color-4); padding-bottom: 10px; margin-bottom: 20px;
}
.section-title { margin: 0; }
.status-text { color: var(--color-5); font-weight: bold; }

.questions-grid {
  display: flex;
  flex-direction: column;
  gap: 25px;
}

.question-card {
  border: 4px solid var(--color-5); padding: 15px;
  background: var(--color-1); box-shadow: 6px 6px 0px var(--color-4);
  transition: all 0.3s;
}

.correct-card { border-color: #10b981; box-shadow: 6px 6px 0px #10b981; }
.wrong-card { border-color: #ef4444; box-shadow: 6px 6px 0px #ef4444; }

.question-header { margin-bottom: 15px; }
.question-header h4 { margin-top: 10px; margin-bottom: 0; line-height: 1.4; font-size: 1.1rem; }
.result-icon { margin-right: 5px; font-size: 1.2rem; }

.type-badge {
  display: inline-block; padding: 2px 8px; font-size: 0.8rem;
  border: 2px solid var(--color-5); color: var(--color-1);
}
.type-badge.single { background-color: #3b82f6; }
.type-badge.multiple { background-color: #ef4444; }
.type-badge.tf { background-color: #10b981; }

.options { display: flex; flex-direction: column; gap: 10px; }

.option-btn {
  background: var(--color-2); color: var(--color-5); border: 2px solid var(--color-5);
  box-shadow: 2px 2px 0px var(--color-5); text-align: left; padding: 10px 15px; margin: 0;
}
.option-btn:hover:not(:disabled) {
  background: var(--color-3); transform: translateX(4px); box-shadow: 0px 2px 0px var(--color-5);
}
.option-btn.selected {
  background: var(--color-4); color: var(--color-1); border-color: var(--color-5);
  box-shadow: inset 4px 4px 0px rgba(0,0,0,0.2); transform: translate(2px, 2px);
}

.option-btn:disabled { cursor: default; opacity: 0.8; }
.option-btn.show-correct { background-color: #10b981 !important; color: white !important; border-color: #047857; }
.option-btn.show-wrong { background-color: #ef4444 !important; color: white !important; border-color: #b91c1c; text-decoration: line-through; }

.multi-check { margin-right: 5px; font-size: 1.2em; }

.explanation-box {
  margin-top: 15px; padding: 15px;
  background-color: var(--color-2); border: 2px dashed var(--color-5);
  font-size: 0.95rem; line-height: 1.5;
}
.exp-title { font-weight: bold; color: #b91c1c; margin-bottom: 5px; }

.slide-down-enter-active, .slide-down-leave-active { transition: all 0.5s ease; }
.slide-down-enter-from, .slide-down-leave-to { opacity: 0; transform: translateY(-10px); }

.review-section { margin-top: 30px; background: var(--color-2); }
.empty-state { color: var(--color-5); opacity: 0.7; padding: 20px; text-align: center; }
.review-list { list-style: none; padding: 0; display: flex; flex-direction: column; gap: 12px; }
.review-item { background: var(--color-1); border: 2px solid var(--color-5); padding: 10px 15px; display: flex; gap: 10px; }
.review-q { font-weight: bold; white-space: nowrap; }
.review-a { color: #b91c1c; font-weight: bold; }

.submit-btn, .reset-btn { width: 100%; font-size: 1.5rem; padding: 15px; margin-top: 20px;}
.submit-btn { background-color: #ef4444; }
.submit-btn:hover { background-color: #dc2626; }
.reset-btn { background-color: var(--color-4); color: var(--color-1); }
</style>