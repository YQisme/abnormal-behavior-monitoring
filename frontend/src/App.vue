<template>
  <router-view v-if="hideSidebar" />
  <el-container v-else class="app-layout">
    <el-aside width="220px" class="side-nav">
      <div class="brand">监测平台</div>
      <el-menu
        :default-active="activePath"
        class="menu"
        router
      >
        <el-menu-item index="/">
          <el-icon><HomeFilled /></el-icon>
          <span>首页</span>
        </el-menu-item>
        <el-menu-item
          v-for="item in visibleMonitorMenus"
          :key="item.path"
          :index="item.path"
        >
          <el-icon><component :is="item.icon" /></el-icon>
          <span>{{ item.label }}</span>
        </el-menu-item>
        <el-sub-menu index="/system-settings">
          <template #title>
            <el-icon><Setting /></el-icon>
            <span>系统设置</span>
          </template>
          <el-menu-item index="/system-settings/layout">排版设置</el-menu-item>
          <el-menu-item index="/system-settings/login">登录设置</el-menu-item>
          <el-menu-item index="/system-settings/restart">重启系统</el-menu-item>
        </el-sub-menu>
      </el-menu>
    </el-aside>
    <el-main class="main-view">
      <router-view />
    </el-main>
  </el-container>
</template>

<script setup>
import { computed, ref, onMounted, onUnmounted } from 'vue'
import { useRoute } from 'vue-router'
import { WarningFilled, UserFilled, CoffeeCup } from '@element-plus/icons-vue'

const MENU_LAYOUT_KEY = 'monitor_menu_layout'

const defaultMonitorMenus = [
  { path: '/zone-alarm', label: '区域报警', icon: WarningFilled, visible: true },
  { path: '/leave-monitor', label: '离岗监测', icon: UserFilled, visible: true },
  { path: '/drowsy-monitor', label: '瞌睡监测', icon: CoffeeCup, visible: true }
]

const route = useRoute()
const monitorMenus = ref([...defaultMonitorMenus])
const hideSidebar = computed(() => route.path === '/login' || route.path === '/')
const activePath = computed(() => route.path)
const visibleMonitorMenus = computed(() => monitorMenus.value.filter(item => item.visible))

const loadMenuLayout = () => {
  try {
    const raw = localStorage.getItem(MENU_LAYOUT_KEY)
    if (!raw) {
      monitorMenus.value = [...defaultMonitorMenus]
      return
    }
    const parsed = JSON.parse(raw)
    if (!Array.isArray(parsed) || parsed.length === 0) {
      monitorMenus.value = [...defaultMonitorMenus]
      return
    }
    const map = new Map(defaultMonitorMenus.map(item => [item.path, item]))
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
    const missing = defaultMonitorMenus
      .filter(item => !normalized.some(m => m.path === item.path))
      .map(item => ({ ...item }))
    monitorMenus.value = [...normalized, ...missing]
  } catch (error) {
    console.error('加载菜单排版配置失败:', error)
    monitorMenus.value = [...defaultMonitorMenus]
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

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
}

#app {
  height: 100vh;
}

.app-layout {
  height: 100vh;
}

.side-nav {
  background: linear-gradient(180deg, #1f3b8f 0%, #2a4fb8 100%);
  color: #fff;
  padding-top: 12px;
}

.brand {
  color: #fff;
  font-size: 22px;
  font-weight: 700;
  text-align: center;
  letter-spacing: 2px;
  margin: 18px 0 16px;
}

.menu {
  border-right: none !important;
  background: transparent !important;
}

.menu .el-menu-item {
  color: rgba(255, 255, 255, 0.88) !important;
}

.menu .el-sub-menu__title,
.menu .el-sub-menu .el-menu-item {
  color: rgba(255, 255, 255, 0.88) !important;
}

.menu .el-menu-item:hover,
.menu .el-menu-item.is-active {
  background: rgba(255, 255, 255, 0.16) !important;
  color: #fff !important;
}

.menu .el-sub-menu__title:hover,
.menu .el-sub-menu.is-active > .el-sub-menu__title,
.menu .el-sub-menu .el-menu-item:hover,
.menu .el-sub-menu .el-menu-item.is-active {
  background: rgba(255, 255, 255, 0.16) !important;
  color: #fff !important;
}

.menu .el-menu--inline {
  background: transparent !important;
}

.main-view {
  padding: 0 !important;
  background: #f5f7fa;
}
</style>

