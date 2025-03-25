<template>
  <div class="min-h-screen flex flex-col relative overflow-hidden">
    <!-- 背景容器 -->
    <div 
      class="absolute inset-0 bg-center bg-no-repeat transition-all duration-500"
      :style="{ backgroundImage: `url('https://api.nsmao.net/api/Img/query?key=kSoR7QHDdC8D0LTdX9LSZkbSW0&sort=acg')` }"
    >
      <!-- 尺寸约束层 -->
      <div class="absolute inset-0 bg-black/20 backdrop-blur-[2px] dark:bg-black/50"></div>
    </div>

    <!-- 内容容器 -->
    <div class="relative z-10 flex-1">
      <div class="p-3 sm:p-8">
        <main class="max-w-7xl mx-auto space-y-8">
          <Header 
            ref="headerRef"
            :title="title"
            :is-refreshing="isRefreshing"
            :is-dark="isDark"
            @refresh="refreshData"
            @toggle-theme="toggleTheme"
          />
          <Stats :monitors="monitors" />
          <Card 
            :monitors="monitors"
            :error="errorMessage"
          />
        </main>
      </div>
      <Footer />
    </div>
  </div>
</template>

<style>
/* 全局样式补充 */
@layer components {
  /* 限制最大缩放比例 */
  .bg-contain-limited {
    background-size: contain;
    background-attachment: local;
    max-width: 100vw;
    max-height: 100vh;
  }

  /* 移动端优化 */
  @media (max-width: 640px) {
    .bg-contain-limited {
      background-attachment: scroll;
      background-size: cover;
    }
  }

  /* 过渡动画 */
  .transition-bg {
    transition: 
      background-image 0.5s cubic-bezier(0.4, 0, 0.2, 1),
      filter 0.3s ease;
  }
}
</style>


<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { fetchMonitorData } from './utils/api'
import Header from './components/Header.vue'
import Stats from './components/Stats.vue'
import Card from './components/Card.vue'
import Footer from './components/Footer.vue'

const title = ref(import.meta.env.VITE_APP_TITLE || '状态监控')
const monitors = ref([])
const isRefreshing = ref(false)
const isDark = ref(false)
const errorMessage = ref('')

// 简化主题相关逻辑
const initTheme = () => {
  isDark.value = localStorage.getItem('theme') === 'dark' || 
    (!localStorage.getItem('theme') && window.matchMedia('(prefers-color-scheme: dark)').matches)
  updateTheme()
}

const updateTheme = () => document.documentElement.classList.toggle('dark', isDark.value)

const toggleTheme = () => {
  isDark.value = !isDark.value
  localStorage.setItem('theme', isDark.value ? 'dark' : 'light')
  updateTheme()
}

// 简化刷新逻辑
const refreshData = async () => {
  if (isRefreshing.value) return
  
  isRefreshing.value = true
  monitors.value = []
  errorMessage.value = ''
  
  const timeoutPromise = new Promise((_, reject) => 
    setTimeout(() => reject(new Error('Timeout')), 30000)
  )

  try {
    monitors.value = await Promise.race([
      fetchMonitorData(),
      timeoutPromise
    ])
  } catch (error) {
    console.error('获取监控数据失败:', error)
    errorMessage.value = error.message === 'Timeout' 
      ? '请求超时，请检查网络连接后重试'
      : '获取监控数据失败，请稍后重试'
  } finally {
    isRefreshing.value = false
  }
}

// 生命周期钩子
onMounted(() => {
  initTheme()
  refreshData()
})
</script>
