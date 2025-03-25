<template>
  <!-- 修改外层容器背景 -->
  <div class="min-h-screen flex flex-col 
    bg-[url('https://api.nsmao.net/api/Img/query?key=kSoR7QHDdC8D0LTdX9LSZkbSW0&sort=acg')] dark:bg-[url('https://api.nsmao.net/api/Img/query?key=kSoR7QHDdC8D0LTdX9LSZkbSW0&sort=acg')]
    bg-cover bg-center bg-no-repeat transition-all duration-300 relative">
    
    <!-- 新增背景遮罩层 -->
    <div class="absolute inset-0 bg-white/20 dark:bg-black/50 z-0"></div>
    
    <!-- 内容容器（包裹原有内容） -->
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
