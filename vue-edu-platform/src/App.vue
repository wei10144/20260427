<template>
  <div class="pixel-cloud cloud-1"></div>
  <div class="pixel-cloud cloud-2"></div>
  <div class="pixel-cloud cloud-3"></div>

  <div class="app-layout">
    <header class="top-navbar">
      <div class="nav-left">
        <div class="brand-logo">互動式課堂管理與測驗平台</div>
      </div>
      
      <div class="nav-center">
        <button :class="['nav-tab', { active: currentTab === 'picker' }]" @click="currentTab = 'picker'">🎲 抽籤機</button>
        <button :class="['nav-tab', { active: currentTab === 'score' }]" @click="currentTab = 'score'">🏆 記分板</button>
        <button :class="['nav-tab', { active: currentTab === 'timer' }]" @click="currentTab = 'timer'">⏱️ 計時器</button>
        <button :class="['nav-tab', { active: currentTab === 'exam' }]" @click="currentTab = 'exam'">📜 測驗系統</button>
        <button :class="['nav-tab', { active: currentTab === 'qa' }]" @click="currentTab = 'qa'">📡 即時問答</button>
      </div>

      </header>

    <main class="main-content">
      <transition name="fade" mode="out-in">
        <RandomPicker v-if="currentTab === 'picker'" />
        <ScoreBoard v-else-if="currentTab === 'score'" />
        <Timer v-else-if="currentTab === 'timer'" />
        <ExamSystem v-else-if="currentTab === 'exam'" />
        <RealtimeQA v-else-if="currentTab === 'qa'" />
      </transition>
    </main>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import RandomPicker from './components/RandomPicker.vue'
import ScoreBoard from './components/ScoreBoard.vue'
import Timer from './components/Timer.vue'
import ExamSystem from './components/ExamSystem.vue'
import RealtimeQA from './components/RealtimeQA.vue'

const currentTab = ref('picker')
</script>

<style scoped>
.app-layout {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  position: relative;
  z-index: 10;
}

/* === 頂部導覽列 === */
.top-navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background-color: var(--color-2);
  border-bottom: 4px solid var(--color-5);
  padding: 10px 20px;
  box-shadow: 0 4px 0 rgba(127, 85, 57, 0.2);
}

.brand-logo {
  font-size: 1.4rem;
  font-weight: bold;
  color: var(--color-5);
  text-shadow: 2px 2px 0 var(--color-1);
  letter-spacing: 1px;
}

.nav-center {
  display: flex;
  gap: 10px;
  flex: 1;
  justify-content: center; /* 讓導覽標籤置中 */
}

.nav-tab {
  background: transparent;
  color: var(--color-5);
  border: 2px solid transparent;
  box-shadow: none;
  padding: 8px 16px;
  font-size: 1.1rem;
}

.nav-tab:hover {
  background: rgba(255, 255, 255, 0.5);
  transform: translateY(-2px);
}

.nav-tab.active {
  background: var(--color-4);
  color: var(--color-1);
  border: 2px solid var(--color-5);
  box-shadow: 2px 2px 0 var(--color-5);
}

/* === 主要內容區 === */
.main-content {
  flex: 1;
  padding: 40px 20px;
  max-width: 1000px;
  margin: 0 auto;
  width: 100%;
}

.fade-enter-active, .fade-leave-active { 
  transition: opacity 0.2s ease, transform 0.2s ease; 
}
.fade-enter-from, .fade-leave-to { 
  opacity: 0; 
  transform: translateY(10px); 
}
</style>