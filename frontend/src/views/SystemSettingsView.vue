<template>
  <div class="system-settings-container">
    <el-container>
      <el-header class="app-header">
        <div class="header-content">
          <h1>系统设置</h1>
        </div>
      </el-header>
      <el-main class="main-content">
        <div class="settings-tabs-card">
          <el-tabs v-model="activeSettingsTab" @tab-change="handleSettingsTabChange">
            <el-tab-pane label="排版设置" name="layout" />
            <el-tab-pane label="登录设置" name="login" />
            <el-tab-pane label="重启系统" name="restart" />
          </el-tabs>
        </div>
        <div class="layout-card" v-if="activeSettingsTab === 'layout'">
          <div class="layout-header">
            <h3>页面板块排版</h3>
            <span>可调整左侧监测模块（含报警事件）的顺序与显示状态</span>
          </div>
          <div class="layout-list">
            <div class="layout-row" v-for="(item, index) in menuLayout" :key="item.path">
              <div class="row-left">
                <span class="row-title">{{ item.label }}</span>
              </div>
              <div class="row-actions">
                <el-switch v-model="item.visible" />
                <el-button size="small" @click="moveUp(index)" :disabled="index === 0">上移</el-button>
                <el-button size="small" @click="moveDown(index)" :disabled="index === menuLayout.length - 1">下移</el-button>
              </div>
            </div>
          </div>
          <div class="layout-footer">
            <el-button @click="resetLayout">恢复默认</el-button>
            <el-button type="primary" @click="saveLayout">保存排版</el-button>
          </div>
        </div>
        <div class="settings-card" v-else>
          <ConfigPanel
            :visible-tabs="configVisibleTabs"
            :initial-tab="activeConfigTab"
          />
        </div>
      </el-main>
    </el-container>
  </div>
</template>

<script setup>
import { computed, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { ElMessage } from 'element-plus'
import ConfigPanel from '../components/ConfigPanel.vue'

const route = useRoute()
const router = useRouter()
const MENU_LAYOUT_KEY = 'monitor_menu_layout'
const defaultLayout = [
  { path: '/zone-alarm', label: '区域报警', visible: true },
  { path: '/leave-monitor', label: '离岗监测', visible: true },
  { path: '/drowsy-monitor', label: '瞌睡监测', visible: true },
  { path: '/alarm-events', label: '报警事件', visible: true }
]
const menuLayout = ref(loadLayout())
const activeSettingsTab = ref(resolveTab(route.params.tab))

const activeConfigTab = computed(() => {
  if (activeSettingsTab.value === 'restart') {
    return 'system'
  }
  return 'login'
})

const configVisibleTabs = computed(() => {
  if (activeSettingsTab.value === 'restart') {
    return ['system']
  }
  return ['login']
})

watch(
  () => route.params.tab,
  (tab) => {
    activeSettingsTab.value = resolveTab(tab)
  }
)

function resolveTab(tab) {
  if (tab === 'layout' || tab === 'login' || tab === 'restart') {
    return tab
  }
  return 'layout'
}

const handleSettingsTabChange = (tabName) => {
  router.push(`/system-settings/${tabName}`)
}

function loadLayout() {
  try {
    const raw = localStorage.getItem(MENU_LAYOUT_KEY)
    if (!raw) return defaultLayout.map(item => ({ ...item }))
    const parsed = JSON.parse(raw)
    if (!Array.isArray(parsed) || parsed.length === 0) {
      return defaultLayout.map(item => ({ ...item }))
    }
    const map = new Map(defaultLayout.map(item => [item.path, item]))
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
    const missing = defaultLayout
      .filter(item => !normalized.some(m => m.path === item.path))
      .map(item => ({ ...item }))
    return [...normalized, ...missing]
  } catch (error) {
    console.error('读取排版配置失败:', error)
    return defaultLayout.map(item => ({ ...item }))
  }
}

const moveUp = (index) => {
  if (index <= 0) return
  const list = [...menuLayout.value]
  ;[list[index - 1], list[index]] = [list[index], list[index - 1]]
  menuLayout.value = list
}

const moveDown = (index) => {
  if (index >= menuLayout.value.length - 1) return
  const list = [...menuLayout.value]
  ;[list[index + 1], list[index]] = [list[index], list[index + 1]]
  menuLayout.value = list
}

const resetLayout = () => {
  menuLayout.value = defaultLayout.map(item => ({ ...item }))
}

const saveLayout = () => {
  localStorage.setItem(MENU_LAYOUT_KEY, JSON.stringify(menuLayout.value))
  window.dispatchEvent(new CustomEvent('menu-layout-updated'))
  ElMessage.success('页面板块排版已保存')
}
</script>

<style scoped>
.system-settings-container {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.app-header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 20px 30px;
  height: auto !important;
}

.header-content h1 {
  margin: 0;
  font-size: 24px;
  font-weight: 600;
}

.main-content {
  padding: 20px;
  background: #f5f7fa;
}

.settings-card {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  padding: 20px;
}

.settings-tabs-card {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  padding: 12px 20px 0;
  margin-bottom: 20px;
}

.layout-card {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  padding: 20px;
  margin-bottom: 20px;
}

.layout-header h3 {
  margin: 0;
  font-size: 18px;
  color: #303133;
}

.layout-header span {
  display: block;
  margin-top: 8px;
  color: #909399;
  font-size: 13px;
}

.layout-list {
  margin-top: 16px;
}

.layout-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 0;
  border-bottom: 1px solid #ebeef5;
}

.layout-row:last-child {
  border-bottom: none;
}

.row-title {
  font-size: 15px;
  color: #303133;
}

.row-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.layout-footer {
  margin-top: 16px;
  display: flex;
  justify-content: flex-end;
  gap: 10px;
}
</style>
