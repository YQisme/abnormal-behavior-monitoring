<template>
  <div class="home-page">
    <div class="hero-card">
      <h1>智能监测系统</h1>
      <p>请选择需要进入的功能模块</p>
      <div class="nav-actions">
        <el-button
          v-for="item in visibleMenus"
          :key="item.path"
          :type="item.buttonType"
          size="large"
          @click="go(item.path)"
        >
          {{ item.label }}
        </el-button>
        <el-empty
          v-if="visibleMenus.length === 0"
          description="当前没有可显示的监测页面，请在系统设置中调整排版"
        />
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, onMounted, onUnmounted, ref } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()
const MENU_LAYOUT_KEY = 'monitor_menu_layout'
const defaultMenus = [
  { path: '/zone-alarm', label: '区域报警', buttonType: 'primary', visible: true },
  { path: '/leave-monitor', label: '离岗监测', buttonType: 'warning', visible: true },
  { path: '/drowsy-monitor', label: '瞌睡监测', buttonType: 'success', visible: true }
]
const monitorMenus = ref([...defaultMenus])

const visibleMenus = computed(() => monitorMenus.value.filter(item => item.visible))

const go = (path) => {
  router.push(path)
}

const loadMenuLayout = () => {
  try {
    const raw = localStorage.getItem(MENU_LAYOUT_KEY)
    if (!raw) {
      monitorMenus.value = [...defaultMenus]
      return
    }
    const parsed = JSON.parse(raw)
    if (!Array.isArray(parsed) || parsed.length === 0) {
      monitorMenus.value = [...defaultMenus]
      return
    }
    const map = new Map(defaultMenus.map(item => [item.path, item]))
    const normalized = parsed
      .map(item => {
        if (!item || !map.has(item.path)) return null
        const base = map.get(item.path)
        return {
          ...base,
          visible: item.visible !== false
        }
      })
      .filter(Boolean)
    const missing = defaultMenus
      .filter(item => !normalized.some(m => m.path === item.path))
      .map(item => ({ ...item }))
    monitorMenus.value = [...normalized, ...missing]
  } catch (error) {
    console.error('首页读取排版配置失败:', error)
    monitorMenus.value = [...defaultMenus]
  }
}

const handleMenuLayoutUpdated = () => {
  loadMenuLayout()
}

onMounted(() => {
  loadMenuLayout()
  window.addEventListener('menu-layout-updated', handleMenuLayoutUpdated)
})

onUnmounted(() => {
  window.removeEventListener('menu-layout-updated', handleMenuLayoutUpdated)
})
</script>

<style scoped>
.home-page {
  min-height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #eef2ff 0%, #f5f7fa 100%);
  padding: 24px;
}

.hero-card {
  width: 100%;
  max-width: 560px;
  text-align: center;
  background: #fff;
  border-radius: 16px;
  box-shadow: 0 12px 32px rgba(30, 64, 175, 0.15);
  padding: 48px 28px;
}

.hero-card h1 {
  font-size: 34px;
  color: #1f2d3d;
  margin-bottom: 12px;
}

.hero-card p {
  color: #5f6b7a;
  font-size: 16px;
  margin-bottom: 28px;
}

.nav-actions {
  display: flex;
  justify-content: center;
  gap: 16px;
  flex-wrap: wrap;
}
</style>
