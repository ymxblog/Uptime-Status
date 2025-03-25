<template>
  <div class="min-h-screen flex flex-col relative overflow-hidden">
    <!-- 独立背景容器 -->
    <div 
      class="absolute inset-0 bg-cover bg-center bg-no-repeat transition-all duration-500"
      :class="{'opacity-0': !backgroundLoaded}"
      :style="{
        backgroundImage: `url(${backgroundUrl})`,
        filter: isDark ? 'brightness(0.6) contrast(1.2)' : 'none'
      }"
    >
      <!-- 智能遮罩层 -->
      <div class="absolute inset-0 bg-gradient-to-b from-black/10 to-black/20 dark:from-black/40 dark:to-black/60"></div>
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

<script setup>
import { ref, watch, onMounted } from 'vue'
import { fetchMonitorData } from './utils/api'
import Header from './components/Header.vue'
import Stats from './components/Stats.vue'
import Card from './components/Card.vue'
import Footer from './components/Footer.vue'

// 响应式数据
const title = ref(import.meta.env.VITE_APP_TITLE || '状态监控')
const monitors = ref([])
const isRefreshing = ref(false)
const isDark = ref(false)
const errorMessage = ref('')
const backgroundUrl = ref('')
const backgroundLoaded = ref(false)

// 主题系统增强
const initTheme = () => {
  isDark.value = localStorage.getItem('theme') === 'dark' || 
    (!localStorage.getItem('theme') && window.matchMedia('(prefers-color-scheme: dark)').matches)
  document.documentElement.classList.toggle('dark', isDark.value)
}

const toggleTheme = () => {
  isDark.value = !isDark.value
  localStorage.setItem('theme', isDark.value ? 'dark' : 'light')
  document.documentElement.classList.toggle('dark', isDark.value)
}

// 图片加载系统
const loadBackground = async () => {
  try {
    const timestamp = new Date().getTime()
    const url = `https://api.nsmao.net/api/Img/query?key=kSoR7QHDdC8D0LTdX9LSZkbSW0&sort=acg&timestamp=${timestamp}`
    
    // 预加载图片
    const img = new Image()
    img.src = url
    await img.decode()
    
    backgroundUrl.value = url
    backgroundLoaded.value = true
  } catch (error) {
    console.error('背景加载失败:', error)
    backgroundUrl.value = '/fallback-bg.jpg'
    backgroundLoaded.value = true
  }
}

// 数据刷新逻辑优化
const refreshData = async () => {
  if (isRefreshing.value) return
  
  isRefreshing.value = true
  monitors.value = []
  errorMessage.value = ''
  
  try {
    const timeout = 15000 // 缩短超时时间
    const controller = new AbortController()
    const timeoutId = setTimeout(() => controller.abort(), timeout)
    
    monitors.value = await fetchMonitorData({ signal: controller.signal })
    clearTimeout(timeoutId)
    
    // 成功时刷新背景
    await loadBackground()
  } catch (error) {
    errorMessage.value = error.name === 'AbortError' 
      ? '请求超时，请检查网络连接后重试'
      : '获取监控数据失败，请稍后重试'
  } finally {
    isRefreshing.value = false
  }
}

// 生命周期钩子增强
onMounted(async () => {
  initTheme()
  await Promise.all([refreshData(), loadBackground()])
})

// 监听主题变化
watch(isDark, (newVal) => {
  document.documentElement.style.setProperty('--theme-transition', 'all 0.5s ease')
})
</script>

<style>
/* 性能优化 */
.absolute {
  backface-visibility: hidden;
  transform: translateZ(0);
}

/* 移动端优化 */
@media (max-width: 640px) {
  .bg-cover {
    background-size: 120% auto;
    background-position: 65% center;
  }
}

/* 过渡优化 */
.transition-all {
  transition-property: background-image, filter, opacity;
  transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
}
</style>
