<template>
  <div v-if="globalFlashRedActive" class="alarm-flash-overlay"></div>
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
import { useRoute, useRouter } from 'vue-router'
import { WarningFilled, UserFilled, CoffeeCup, BellFilled } from '@element-plus/icons-vue'
import { io } from 'socket.io-client'
import { ElMessageBox } from 'element-plus'

const MENU_LAYOUT_KEY = 'monitor_menu_layout'

const defaultMonitorMenus = [
  { path: '/zone-alarm', label: '区域报警', icon: WarningFilled, visible: true },
  { path: '/leave-monitor', label: '离岗监测', icon: UserFilled, visible: true },
  { path: '/drowsy-monitor', label: '瞌睡监测', icon: CoffeeCup, visible: true },
  { path: '/alarm-events', label: '报警事件', icon: BellFilled, visible: true }
]

const route = useRoute()
const router = useRouter()
const monitorMenus = ref([...defaultMonitorMenus])
const hideSidebar = computed(() => route.path === '/login' || route.path === '/')
const activePath = computed(() => route.path)
const visibleMonitorMenus = computed(() => monitorMenus.value.filter(item => item.visible))
const globalFlashRedActive = ref(false)
let globalFlashTimer = null
let alarmSocket = null
let globalAlarmHandling = false
const globalAlarmQueue = []
let speechPrimed = false

const toChineseDrowsyState = (state) => {
  if (state === 'Drowsy + Yawning') return '疑似瞌睡并打哈欠'
  if (state === 'Drowsy') return '疑似瞌睡'
  if (state === 'Yawning') return '频繁打哈欠'
  return '状态异常'
}

const getProfileDisplayName = (profile) => {
  if (profile === 'offpost_monitor') return '离岗监测'
  if (profile === 'drowsy_monitor') return '瞌睡监测'
  if (profile === 'zone_alarm') return '区域报警'
  return '未知来源'
}

const normalizeObjectName = (name) => {
  const text = String(name || '').trim()
  if (!text) return '目标'
  if (/[A-Za-z]/.test(text) && !/[\u4e00-\u9fa5]/.test(text)) return '目标'
  return text
}

const buildGlobalAlarmText = (alarm) => {
  const sourceTag = `【${getProfileDisplayName(alarm?.profile)}】`
  const zoneName = alarm?.zone_name ? `${alarm.zone_name}` : ''
  if (alarm?.alarm_type === 'offpost_absence') {
    const duration = alarm?.absence_duration !== undefined ? `${Number(alarm.absence_duration).toFixed(1)}秒` : '持续无人'
    return `${sourceTag}${zoneName}已${duration}未检测到人员，疑似离岗`
  }
  if (alarm?.alarm_type === 'offpost_recovery') {
    const duration = alarm?.absence_duration !== undefined ? `${Number(alarm.absence_duration).toFixed(1)}秒` : '一段时间'
    return `${sourceTag}${zoneName}已离岗${duration}`
  }
  if (alarm?.alarm_type === 'drowsy') {
    return `${sourceTag}${zoneName}瞌睡报警，${toChineseDrowsyState(alarm?.drowsy_state)}`
  }
  return `${sourceTag}${normalizeObjectName(alarm?.class_name_cn || alarm?.object_name)}进入监控区域${zoneName}`
}

const buildAlarmEventType = (alarmType) => {
  if (alarmType === 'offpost_absence' || alarmType === 'offpost_recovery') return 'offpost'
  if (alarmType === 'drowsy') return 'drowsy'
  if (alarmType === 'zone') return 'zone'
  return 'unknown'
}

const buildAlarmEventId = (alarm) => {
  const eventFile = alarm?.event_video || alarm?.event_image
  if (!eventFile) return ''
  const stem = String(eventFile).replace(/\.[^.]+$/, '')
  if (!stem) return ''
  return `${buildAlarmEventType(alarm?.alarm_type)}:${stem}`
}

const goToAlarmReplay = (alarm) => {
  const eventType = buildAlarmEventType(alarm?.alarm_type)
  const query = {
    view: 'timeline',
    event_type: eventType === 'unknown' ? 'all' : eventType
  }
  const eventId = buildAlarmEventId(alarm)
  if (eventId) {
    query.eventId = eventId
  }
  router.push({ path: '/alarm-events', query })
}

const speakAlarm = (text) => {
  if (!text || typeof window === 'undefined' || !window.speechSynthesis) return
  try {
    // 首次播报前预热一次语音引擎，降低首次不发声概率
    if (!speechPrimed) {
      window.speechSynthesis.getVoices()
      speechPrimed = true
    }
    const utterance = new SpeechSynthesisUtterance(text)
    utterance.lang = 'zh-CN'
    utterance.rate = 1
    utterance.pitch = 1
    utterance.volume = 1
    window.speechSynthesis.cancel()
    // 让 cancel 和 speak 分两个任务执行，避免部分浏览器吞掉本次播报
    setTimeout(() => {
      try {
        window.speechSynthesis.speak(utterance)
      } catch (error) {
        console.warn('全局语音播报失败:', error)
      }
    }, 20)
  } catch (error) {
    console.warn('全局语音播报失败:', error)
  }
}

const openGlobalFlash = () => {
  globalFlashRedActive.value = true
}

const closeGlobalFlash = () => {
  globalFlashRedActive.value = false
  if (globalFlashTimer) {
    clearTimeout(globalFlashTimer)
    globalFlashTimer = null
  }
}

const closeGlobalFlashLater = (ms = 5000) => {
  if (globalFlashTimer) {
    clearTimeout(globalFlashTimer)
  }
  globalFlashTimer = setTimeout(() => {
    globalFlashRedActive.value = false
    globalFlashTimer = null
  }, ms)
}

const processGlobalAlarmQueue = async () => {
  if (globalAlarmHandling || globalAlarmQueue.length === 0) return
  globalAlarmHandling = true
  const alarm = globalAlarmQueue.shift()
  const alarmText = buildGlobalAlarmText(alarm)
  try {
    if (alarm?.sound_light_alarm_enabled !== false) {
      openGlobalFlash()
      speakAlarm(alarmText)
    }
    if (alarm?.popup_alarm_enabled !== false) {
      await ElMessageBox.confirm(alarmText, '报警提示', {
        confirmButtonText: '关闭报警',
        cancelButtonText: '查看回放',
        distinguishCancelAndClose: true,
        showCancelButton: true,
        type: 'warning',
        closeOnClickModal: false,
        closeOnPressEscape: false,
        showClose: false,
        customClass: 'alarm-confirm-dialog'
      })
      if (alarm?.sound_light_alarm_enabled !== false) {
        closeGlobalFlash()
      }
    } else if (alarm?.sound_light_alarm_enabled !== false) {
      // 无弹窗时给闪红保留一个可感知时长
      closeGlobalFlashLater(5000)
    }
  } catch (error) {
    if (error === 'cancel') {
      goToAlarmReplay(alarm)
      if (alarm?.sound_light_alarm_enabled !== false) {
        closeGlobalFlash()
      }
    }
    console.warn('全局报警确认中断:', error)
  } finally {
    // 不在这里强制 cancel，避免用户快速关闭弹窗时把整段播报直接打断
    globalAlarmHandling = false
    processGlobalAlarmQueue()
  }
}

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
  const socketUrl = import.meta.env.DEV
    ? (import.meta.env.VITE_API_URL || `http://${window.location.hostname}:5000`)
    : window.location.origin
  alarmSocket = io(socketUrl, {
    transports: ['polling'],
    reconnection: true,
    reconnectionDelay: 1000,
    reconnectionAttempts: 5
  })
  alarmSocket.on('alarm', (data) => {
    if (!data) return
    // 登录页/首页不弹全局报警，避免影响登录流程
    if (route.path === '/login' || route.path === '/') return
    globalAlarmQueue.push(data)
    processGlobalAlarmQueue()
  })
})

onUnmounted(() => {
  window.removeEventListener('menu-layout-updated', handleMenuLayoutUpdated)
  if (alarmSocket) {
    alarmSocket.disconnect()
  }
  if (typeof window !== 'undefined' && window.speechSynthesis) {
    window.speechSynthesis.cancel()
  }
  closeGlobalFlash()
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

.alarm-flash-overlay {
  position: fixed;
  inset: 0;
  pointer-events: none;
  display: flex;
  justify-content: center;
  align-items: center;
  background: rgba(255, 0, 0, 0.2);
  z-index: 9999;
  animation: alarm-red-blink 0.8s ease-in-out infinite;
}

@keyframes alarm-red-blink {
  0% {
    background: rgba(255, 0, 0, 0.18);
  }
  50% {
    background: rgba(255, 0, 0, 0.55);
  }
  100% {
    background: rgba(255, 0, 0, 0.18);
  }
}
</style>

