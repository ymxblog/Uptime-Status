<template>
  <!-- 修改外层容器 -->
  <div class="min-h-screen w-full flex flex-col 
    bg-[url('https://images.418121.xyz/random?dir=blog/camera/&&type=img')] 
    dark:bg-[url('https://images.418121.xyz/file/lqlb/1760698332227_1288651.png')]
    bg-cover bg-center bg-no-repeat bg-fixed  <!-- 新增 bg-fixed -->
    transition-all duration-300 relative">
    
    <!-- 优化遮罩层 -->
    <div class="absolute inset-0 bg-white/15 dark:bg-black/60 backdrop-blur-[2px] z-0"></div>
    
    <!-- 内容容器保持不变 -->
    <div class="relative z-10 flex-1">
      <div class="p-3 sm:p-8">
        <main class="max-w-7xl mx-auto space-y-8">
        <Header :title="title" :is-refreshing="isRefreshing" :is-dark="isDark" v-model:sort="sort"
          @refresh="load" @toggle-theme="toggleTheme" @toggle-language="toggleLanguage" />
          <Stats :monitors="monitors" />
        <Card :monitors="monitors" :sort="sort" :error="error" :refreshing="isRefreshing" @update-monitor="onPatch" />
        </main>
      </div>
      <Footer />
    </div>
  </div>
</template>



<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { fetchMonitorData, writeCache, readCache, isRateLimit, waitRetry } from './utils/api'
import Header from './components/Header.vue'
import Stats from './components/Stats.vue'
import Card from './components/Card.vue'
import Footer from './components/Footer.vue'

const { t, locale } = useI18n()
const title = ref(import.meta.env.VITE_APP_TITLE || t('common.title'))
const monitors = ref([])
const isRefreshing = ref(false)
const isDark = ref(false)
const error = ref('')
const loadSort = () => {
  const raw = localStorage.getItem('monitorSort')
  if (!raw) return { key: 'friendlyName', order: 'asc' }
  if (raw.includes(':')) {
    const [key, order] = raw.split(':')
    return { key: key || 'friendlyName', order: order === 'desc' ? 'desc' : 'asc' }
  }
  return { key: raw, order: raw === 'createDateTime' ? 'desc' : 'asc' }
}

const sort = ref(loadSort())

watch(sort, (v) => localStorage.setItem('monitorSort', `${v.key}:${v.order}`), { deep: true })
watch(locale, () => { title.value = import.meta.env.VITE_APP_TITLE || t('common.title') })

const toggleLanguage = () => {
  locale.value = locale.value === 'zh-CN' ? 'en-US' : 'zh-CN'
  localStorage.setItem('locale', locale.value)
}

const initTheme = () => {
  isDark.value = localStorage.getItem('theme') === 'dark' ||
    (!localStorage.getItem('theme') && matchMedia('(prefers-color-scheme: dark)').matches)
  document.documentElement.classList.toggle('dark', isDark.value)
}

const toggleTheme = () => {
  isDark.value = !isDark.value
  localStorage.setItem('theme', isDark.value ? 'dark' : 'light')
  document.documentElement.classList.toggle('dark', isDark.value)
}

let loadId = 0

const load = async (force = false) => {
  if (isRefreshing.value) return
  const id = ++loadId
  isRefreshing.value = true
  if (force) error.value = ''
  let useForce = force

  while (id === loadId) {
    try {
      monitors.value = await fetchMonitorData({ force: useForce })
      error.value = ''
      break
    } catch (e) {
      if (id !== loadId) break
      const stale = readCache(true)
      if (stale?.length) monitors.value = stale
      if (!isRateLimit(e)) { error.value = t('error.fetchFailed'); break }
      isRefreshing.value = false
      await waitRetry(e.retryAfter || 60, (s) => {
        if (id === loadId) {
          error.value = monitors.value.length
            ? t('error.rateLimitRetry', { seconds: s })
            : t('error.rateLimit', { seconds: s })
        }
      })
      if (id !== loadId) break
      isRefreshing.value = true
      useForce = true
    }
  }
  if (id === loadId) isRefreshing.value = false
}

const onPatch = (m) => {
  const i = monitors.value.findIndex((x) => x.id === m.id)
  if (i >= 0) { monitors.value[i] = m; writeCache(monitors.value) }
}

onMounted(() => { initTheme(); load() })
onUnmounted(() => { loadId++ })
</script>
