<template>
  <div class="app-container">
    <el-container>
      <!-- 头部 -->
      <el-header class="app-header">
        <div class="header-content">
          <h1>{{ pageTitle }}</h1>
          <div class="status-bar">
            <el-tag :type="connectionStatus === 'connected' ? 'success' : 'danger'" effect="dark">
              {{ connectionStatus === 'connected' ? '已连接' : '未连接' }}
            </el-tag>
            <el-tag :type="zonesCount > 0 ? 'success' : 'info'" effect="dark">
              {{ zonesCount > 0 ? `已设置 ${zonesCount} 个区域` : '未设置区域' }}
            </el-tag>
            <el-button type="primary" size="small" @click="goToMonitor">视频监控</el-button>
            <el-button type="danger" size="small" @click="handleLogout" v-if="loginEnabled">登出</el-button>
          </div>
        </div>
      </el-header>

      <!-- 主内容区 -->
      <el-main class="main-content">
        <div v-if="flashRedActive" class="alarm-flash-overlay">
          <div v-if="flashConfirmVisible" class="flash-confirm-card">
            <div class="flash-confirm-title">声光报警</div>
            <div class="flash-confirm-text">{{ flashConfirmText }}</div>
            <div class="flash-confirm-actions">
              <el-button @click="goToAlarmReplay">查看回放</el-button>
              <el-button type="danger" @click="confirmFlashAlarm">关闭报警</el-button>
            </div>
          </div>
        </div>
        <!-- 上方：视频 + 监控信息 -->
        <el-row :gutter="20" class="video-row">
          <el-col :span="16">
            <VideoPanel
              :key="`video-panel-${alarmApiPrefix}`"
              ref="videoPanelRef"
              :zones="zones"
              :api-prefix="alarmApiPrefix"
              :panel-title="videoPanelTitle"
              @zones-updated="handleZonesUpdated"
            />
          </el-col>
          <el-col :span="8">
            <el-card shadow="hover" class="info-card">
              <template #header>
                <div class="card-header">
                  <span class="card-title">监控信息</span>
                </div>
              </template>
              <el-tabs v-model="activeInfoTab" type="border-card" class="info-tabs">
                <el-tab-pane label="报警记录" name="alarm">
                  <div class="alarm-tab-wrap">
                    <div class="alarm-header">
                      <el-button size="small" type="primary" @click="clearAlarmMqtt" :loading="clearAlarmMqttLoading">清除报警信息</el-button>
                    </div>
                    <AlarmList :alarms="alarms" />
                  </div>
                </el-tab-pane>
                <el-tab-pane label="检测信息" name="detection">
                  <DetectionInfo :detections="detections" />
                </el-tab-pane>
              </el-tabs>
            </el-card>
          </el-col>
        </el-row>

        <!-- 下方：区域管理、日志、设置 -->
        <div class="bottom-nav-bar">
          <el-tabs v-model="activeTopTab" type="card" class="top-tabs">
            <el-tab-pane :label="zoneTabLabel" name="zones">
              <div class="tab-content">
                <ZoneManager
                  :zones="zones"
                  :api-prefix="alarmApiPrefix"
                  @zone-selected="handleZoneSelected"
                  @start-drawing="handleStartDrawing"
                  @zone-updated="handleZoneUpdated"
                />
              </div>
            </el-tab-pane>
            <el-tab-pane label="日志" name="logs">
              <div class="tab-content">
                <el-tabs v-model="activeLogTab" type="border-card" class="log-tabs">
                  <el-tab-pane label="操作日志" name="operation">
                    <div class="log-container">
                      <div class="log-header">
                        <el-button size="small" @click="clearOperationLogs">清空日志</el-button>
                        <el-checkbox v-model="autoScrollOperationLogs" size="small" style="margin-left: 10px">
                          自动滚动
                        </el-checkbox>
                      </div>
                      <LogPanel :logs="operationLogs" :auto-scroll="autoScrollOperationLogs" />
                    </div>
                  </el-tab-pane>
                  <el-tab-pane label="后端日志" name="backend">
                    <div class="log-container">
                      <div class="log-header">
                        <el-button size="small" @click="clearBackendLogs">清空日志</el-button>
                        <el-checkbox v-model="autoScrollBackendLogs" size="small" style="margin-left: 10px">
                          自动滚动
                        </el-checkbox>
                      </div>
                      <LogPanel :logs="backendLogs" :auto-scroll="autoScrollBackendLogs" />
                    </div>
                  </el-tab-pane>
                  <el-tab-pane label="推理日志" name="yolo">
                    <div class="log-container">
                      <div class="log-header">
                        <el-button size="small" @click="clearYoloLogs">清空日志</el-button>
                        <el-checkbox v-model="autoScrollYoloLogs" size="small" style="margin-left: 10px">
                          自动滚动
                        </el-checkbox>
                      </div>
                      <LogPanel :logs="yoloLogs" :auto-scroll="autoScrollYoloLogs" />
                    </div>
                  </el-tab-pane>
                </el-tabs>
              </div>
            </el-tab-pane>
            <el-tab-pane label="设置" name="config">
              <div class="tab-content">
                <ConfigPanel
                  :alarm-api-prefix="alarmApiPrefix"
                  :visible-tabs="['model', 'video', 'classes', 'display', 'occlusion', 'mqtt', 'alarm']"
                  @model-changed="handleModelChanged"
                  @video-changed="handleVideoChanged"
                  @classes-changed="handleClassesChanged"
                  @display-changed="handleDisplayChanged"
                  @alarm-changed="handleAlarmChanged"
                />
              </div>
            </el-tab-pane>
          </el-tabs>
        </div>
      </el-main>
    </el-container>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { io } from 'socket.io-client'
import { ElMessage, ElMessageBox } from 'element-plus'
import axios from 'axios'
import ConfigPanel from '../components/ConfigPanel.vue'
import VideoPanel from '../components/VideoPanel.vue'
import AlarmList from '../components/AlarmList.vue'
import DetectionInfo from '../components/DetectionInfo.vue'
import LogPanel from '../components/LogPanel.vue'
import ZoneManager from '../components/ZoneManager.vue'

const router = useRouter()
const route = useRoute()
const resolveAlarmApiPrefixByPath = (path) => {
  if (path === '/leave-monitor') return '/api/offpost'
  if (path === '/drowsy-monitor') return '/api/drowsy'
  return '/api'
}
const isLeaveMonitorMode = ref(false)
const isDrowsyMonitorMode = ref(false)
const pageTitle = ref('人员检测与区域报警系统')
const videoPanelTitle = ref('区域报警')
const zoneTabLabel = ref('区域管理')
const alarmApiPrefix = ref(resolveAlarmApiPrefixByPath(route.path))

// 状态
const connectionStatus = ref('disconnected')
const zones = ref([])
const zonesCount = ref(0)
const alarms = ref([])
const detections = ref([])
const createModeLogs = () => ({
  backend: [],
  yolo: [],
  operation: []
})
const logsByMode = ref({
  zone: createModeLogs(),
  offpost: createModeLogs(),
  drowsy: createModeLogs()
})
const autoScrollBackendLogs = ref(true)
const autoScrollYoloLogs = ref(true)
const autoScrollOperationLogs = ref(true)
const activeTopTab = ref('zones')  // 顶部标签：zones/logs/config
const activeLogTab = ref('operation')  // 日志子标签：operation/backend/yolo
const activeInfoTab = ref('alarm')  // 信息标签：alarm/detection
const videoPanelRef = ref(null)
const loginEnabled = ref(true)  // 是否启用登录
const clearAlarmMqttLoading = ref(false)
const flashRedActive = ref(false)
const flashConfirmVisible = ref(false)
const flashConfirmText = ref('')
const flashConfirmAlarm = ref(null)
let flashConfirmResolver = null
let alarmActionRunning = false
const pendingAlarmQueue = []

// Socket.IO 连接
let socket = null

const getCurrentModeKey = () => {
  if (route.path === '/leave-monitor') {
    return 'offpost'
  }
  if (route.path === '/drowsy-monitor') {
    return 'drowsy'
  }
  return 'zone'
}

const getCurrentModeLogs = () => logsByMode.value[getCurrentModeKey()]
const backendLogs = computed(() => getCurrentModeLogs().backend)
const yoloLogs = computed(() => getCurrentModeLogs().yolo)
const operationLogs = computed(() => getCurrentModeLogs().operation)

const getModeKeyFromProfile = (profile) => {
  if (profile === 'offpost_monitor') {
    return 'offpost'
  }
  if (profile === 'drowsy_monitor') {
    return 'drowsy'
  }
  return 'zone'
}

const isInferenceBackendLog = (data) => {
  const message = String(data?.message || '').toLowerCase()
  if (!message) return false
  // 兼容后端将推理相关信息写在 backend_logger 的情况
  return [
    'yolo',
    'ultralytics',
    '推理',
    '检测',
    'imgsz',
    'model',
    'track'
  ].some((token) => message.includes(token))
}

const refreshPageMode = () => {
  isLeaveMonitorMode.value = route.path === '/leave-monitor'
  isDrowsyMonitorMode.value = route.path === '/drowsy-monitor'
  if (isLeaveMonitorMode.value) {
    pageTitle.value = '人员检测与离岗监测系统'
    videoPanelTitle.value = '离岗监测'
    zoneTabLabel.value = '离岗区域管理'
    alarmApiPrefix.value = resolveAlarmApiPrefixByPath(route.path)
    return
  }
  if (isDrowsyMonitorMode.value) {
    pageTitle.value = '人员检测与瞌睡监测系统'
    videoPanelTitle.value = '瞌睡监测'
    zoneTabLabel.value = '瞌睡区域管理'
    alarmApiPrefix.value = resolveAlarmApiPrefixByPath(route.path)
    return
  }
  pageTitle.value = '人员检测与区域报警系统'
  videoPanelTitle.value = '区域报警'
  zoneTabLabel.value = '区域管理'
  alarmApiPrefix.value = resolveAlarmApiPrefixByPath(route.path)
}

const activateCurrentProfile = async () => {
  try {
    await axios.post(`${alarmApiPrefix.value}/activate_profile`)
  } catch (error) {
    console.error('激活监测模式失败:', error)
  }
}

onMounted(() => {
  refreshPageMode()
  activateCurrentProfile()
  // 检查登录状态
  checkLoginStatus()

  // 根据环境配置 Socket.IO 连接地址
  const socketUrl = import.meta.env.DEV 
    ? (import.meta.env.VITE_API_URL || `http://${window.location.hostname}:5000`)
    : window.location.origin
  
  socket = io(socketUrl, {
    // 在当前 Flask/threading 部署下优先使用 long-polling，避免 websocket 握手 500
    transports: ['polling'],
    reconnection: true,
    reconnectionDelay: 1000,
    reconnectionAttempts: 5
  })

  socket.on('connect', () => {
    console.log('已连接到服务器', socket.id)
    connectionStatus.value = 'connected'
    // 连接后加载区域列表
    handleZonesUpdated()
  })

  socket.on('disconnect', () => {
    console.log('与服务器断开连接')
    connectionStatus.value = 'disconnected'
  })

  socket.on('frame', (data) => {
    if (!data) {
      console.warn('收到空的 frame 数据')
      return
    }
    
    if (videoPanelRef.value) {
      videoPanelRef.value.updateFrame(data)
    } else {
      console.warn('videoPanelRef 未初始化')
    }
    
    // 更新区域状态
    if (data.zones) {
      zones.value = data.zones
      zonesCount.value = data.zones.length
    }
    
    // 更新检测信息
    detections.value = data.detections || []
  })
  
  socket.on('connect_error', (error) => {
    console.error('Socket.IO 连接错误:', error)
    connectionStatus.value = 'disconnected'
  })

  socket.on('alarm', (data) => {
    // 离岗监测模式下忽略“有人进入区域”的报警，只保留持续无人报警
    if (isLeaveMonitorMode.value && (data?.alarm_type === 'zone' || !data?.alarm_type)) {
      return
    }
    console.log('报警:', data)
    alarms.value.unshift(data)
    if (alarms.value.length > 20) {
      alarms.value.pop()
    }
    runAlarmActions(data)
  })
  
  socket.on('alarm_cleared', () => {
    alarms.value = []
  })

  socket.on('log', (data) => {
    // 系统日志按日志自带的监测模式分流，避免离岗/瞌睡/区域报警日志重叠
    const targetModeKey = getModeKeyFromProfile(data?.profile)
    const targetModeLogs = logsByMode.value[targetModeKey]
    const loggerName = String(data?.logger || '').toLowerCase()
    if (loggerName === 'backend' && isInferenceBackendLog(data)) {
      targetModeLogs.yolo.push(data)
      if (targetModeLogs.yolo.length > 1000) {
        targetModeLogs.yolo.shift()
      }
      return
    }

    if (loggerName === 'backend') {
      targetModeLogs.backend.push(data)
      if (targetModeLogs.backend.length > 1000) {
        targetModeLogs.backend.shift()
      }
      return
    }

    if (loggerName === 'yolo' || loggerName === 'ultralytics') {
      targetModeLogs.yolo.push(data)
      if (targetModeLogs.yolo.length > 1000) {
        targetModeLogs.yolo.shift()
      }
      return
    }

    // 未知来源默认归入后端日志，避免日志丢失
    targetModeLogs.backend.push(data)
    if (targetModeLogs.backend.length > 1000) {
      targetModeLogs.backend.shift()
    }
  })
})

onUnmounted(() => {
  if (socket) {
    socket.disconnect()
  }
  closeFlashRed()
  if (flashConfirmResolver) {
    flashConfirmResolver()
    flashConfirmResolver = null
  }
  if (typeof window !== 'undefined' && window.speechSynthesis) {
    window.speechSynthesis.cancel()
  }
})

watch(() => route.path, () => {
  refreshPageMode()
  activateCurrentProfile()
  handleZonesUpdated()
})

const handleZonesUpdated = async () => {
  // 重新加载区域列表
  try {
    const res = await axios.get(`${alarmApiPrefix.value}/zones`)
    if (res.data.zones) {
      zones.value = res.data.zones
      zonesCount.value = res.data.zones.length
    }
  } catch (error) {
    console.error('加载区域失败:', error)
  }
}

const toChineseDrowsyState = (state) => {
  if (state === 'Drowsy + Yawning') return '疑似瞌睡并打哈欠'
  if (state === 'Drowsy') return '疑似瞌睡'
  if (state === 'Yawning') return '频繁打哈欠'
  return '状态异常'
}

const normalizeObjectName = (name) => {
  const text = String(name || '').trim()
  if (!text) return '目标'
  if (/[A-Za-z]/.test(text) && !/[\u4e00-\u9fa5]/.test(text)) {
    return '目标'
  }
  return text
}

const buildAlarmText = (alarm) => {
  const zoneName = alarm?.zone_name ? `${alarm.zone_name}` : ''
  if (alarm?.alarm_type === 'offpost_absence') {
    const duration = alarm?.absence_duration !== undefined ? `${Number(alarm.absence_duration).toFixed(1)}秒` : '持续无人'
    return `${zoneName}已${duration}未检测到人员，疑似离岗`
  }
  if (alarm?.alarm_type === 'offpost_recovery') {
    const duration = alarm?.absence_duration !== undefined ? `${Number(alarm.absence_duration).toFixed(1)}秒` : '一段时间'
    return `${zoneName}已离岗${duration}`
  }
  if (alarm?.alarm_type === 'drowsy') {
    return `${zoneName}瞌睡报警，${toChineseDrowsyState(alarm?.drowsy_state)}`
  }
  return `${normalizeObjectName(alarm?.class_name_cn || alarm?.object_name)}进入监控区域${zoneName}`
}

const triggerFlashRed = (text, needManualConfirm, alarm = null) => {
  flashRedActive.value = true
  flashConfirmText.value = text
  flashConfirmVisible.value = !!needManualConfirm
  flashConfirmAlarm.value = needManualConfirm ? alarm : null
}

const closeFlashRed = () => {
  flashRedActive.value = false
  flashConfirmVisible.value = false
  flashConfirmText.value = ''
  flashConfirmAlarm.value = null
}

const waitFlashConfirm = () => new Promise((resolve) => {
  flashConfirmResolver = resolve
})

const confirmFlashAlarm = () => {
  if (flashConfirmResolver) {
    flashConfirmResolver()
    flashConfirmResolver = null
  }
  closeFlashRed()
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

const parseAlarmTimestamp = (alarm) => {
  const raw = String(alarm?.time || '').trim()
  if (!raw) return 0
  const ms = Date.parse(raw.replace(' ', 'T'))
  return Number.isNaN(ms) ? 0 : ms / 1000
}

const findFallbackEventId = async (alarm) => {
  const eventType = buildAlarmEventType(alarm?.alarm_type)
  const queryEventType = eventType === 'unknown' ? 'all' : eventType
  try {
    const res = await axios.get('/api/alarm-events', {
      params: {
        limit: 80,
        event_type: queryEventType
      }
    })
    if (!res.data?.success) return ''
    const grouped = res.data?.grouped_timeline || {}
    const events = Object.values(grouped).flat()
    if (!Array.isArray(events) || events.length === 0) return ''
    const alarmTs = parseAlarmTimestamp(alarm)
    const alarmZone = String(alarm?.zone_name || '').trim()
    const alarmTrack = String(alarm?.track_id ?? '').trim()
    const sorted = [...events].sort((a, b) => {
      const aDoc = a?.document || {}
      const bDoc = b?.document || {}
      const aZone = String(aDoc.zone_name || '').trim()
      const bZone = String(bDoc.zone_name || '').trim()
      const aTrack = String(aDoc.track_id ?? '').trim()
      const bTrack = String(bDoc.track_id ?? '').trim()
      const aTs = Number(a?.occurred_at || 0)
      const bTs = Number(b?.occurred_at || 0)
      const aScore = (alarmZone && aZone === alarmZone ? 2 : 0) + (alarmTrack && aTrack === alarmTrack ? 2 : 0)
      const bScore = (alarmZone && bZone === alarmZone ? 2 : 0) + (alarmTrack && bTrack === alarmTrack ? 2 : 0)
      if (aScore !== bScore) return bScore - aScore
      if (!alarmTs) return bTs - aTs
      return Math.abs(aTs - alarmTs) - Math.abs(bTs - alarmTs)
    })
    return sorted[0]?.event_id || ''
  } catch (error) {
    console.warn('匹配报警事件失败:', error)
    return ''
  }
}

const goToAlarmReplay = async () => {
  const alarm = flashConfirmAlarm.value
  let eventId = buildAlarmEventId(alarm)
  const eventType = buildAlarmEventType(alarm?.alarm_type)
  const queryEventType = eventType === 'unknown' ? 'all' : eventType
  if (!eventId) {
    eventId = await findFallbackEventId(alarm)
  }
  const query = {
    view: 'timeline',
    event_type: queryEventType
  }
  if (eventId) {
    query.eventId = eventId
  } else {
    ElMessage.warning('未匹配到对应回放，已跳转到报警事件列表')
  }
  router.push({ path: '/alarm-events', query })
}

const speakAlarm = (text) => {
  if (!text || typeof window === 'undefined' || !window.speechSynthesis) return
  try {
    const utterance = new SpeechSynthesisUtterance(text)
    utterance.lang = 'zh-CN'
    utterance.rate = 1
    utterance.pitch = 1
    utterance.volume = 1
    // 打断上一条，优先播报最新报警
    window.speechSynthesis.cancel()
    window.speechSynthesis.speak(utterance)
  } catch (error) {
    console.warn('语音播报失败:', error)
  }
}

const runAlarmActions = (alarm) => {
  pendingAlarmQueue.push(alarm)
  processAlarmQueue()
}

const processAlarmQueue = async () => {
  if (alarmActionRunning || pendingAlarmQueue.length === 0) return
  alarmActionRunning = true

  const alarm = pendingAlarmQueue.shift()
  const popupEnabled = alarm?.popup_alarm_enabled !== false
  const soundLightEnabled = alarm?.sound_light_alarm_enabled !== false
  const alarmText = buildAlarmText(alarm)

  try {
    if (soundLightEnabled) {
      triggerFlashRed(alarmText, !popupEnabled, alarm)
      speakAlarm(alarmText)
    }

    if (popupEnabled) {
      await ElMessageBox.confirm(alarmText, '报警提示', {
        confirmButtonText: '关闭报警',
        cancelButtonText: '查看回放',
        distinguishCancelAndClose: true,
        type: 'warning',
        closeOnClickModal: false,
        closeOnPressEscape: false,
        showClose: false,
        customClass: 'alarm-confirm-dialog'
      })
      if (soundLightEnabled) {
        closeFlashRed()
      }
    } else if (soundLightEnabled) {
      await waitFlashConfirm()
    }
  } catch (error) {
    if (error === 'cancel') {
      await goToAlarmReplay()
      if (soundLightEnabled) {
        closeFlashRed()
      }
    } else {
      console.warn('报警确认中断:', error)
    }
  } finally {
    if (typeof window !== 'undefined' && window.speechSynthesis) {
      window.speechSynthesis.cancel()
    }
    alarmActionRunning = false
    processAlarmQueue()
  }
}

const handleZoneSelected = (zoneId) => {
  // 区域选择处理（可选）
}

const handleStartDrawing = (zone = null) => {
  // 开始绘制区域
  if (videoPanelRef.value && videoPanelRef.value.startDrawing) {
    videoPanelRef.value.startDrawing(zone)
    if (zone) {
      addOperationLog(`开始编辑区域: ${zone.name}`, 'INFO')
    } else {
      addOperationLog('开始绘制新区域', 'INFO')
    }
  }
}

const handleZoneUpdated = () => {
  handleZonesUpdated()
  addOperationLog('区域配置已更新', 'INFO')
}

const clearBackendLogs = () => {
  getCurrentModeLogs().backend = []
}

const clearYoloLogs = () => {
  getCurrentModeLogs().yolo = []
}

// 清除报警信息：向 MQTT 发送 isOffline/isOccluded/hasPeople 的 0 消息
const clearAlarmMqtt = async () => {
  clearAlarmMqttLoading.value = true
  try {
    const res = await axios.post('/api/mqtt/clear_alarm')
    if (res.data.success) {
      // 无论是否收到 WebSocket 广播，当前页面都立即清空
      alarms.value = []
      ElMessage.success(res.data.message || '报警记录已清除')
    } else {
      ElMessage.warning(res.data.message || '操作失败')
    }
  } catch (error) {
    console.error('清除报警信息失败:', error)
    ElMessage.error(error.response?.data?.message || '请求失败')
  } finally {
    clearAlarmMqttLoading.value = false
  }
}

const clearOperationLogs = () => {
  getCurrentModeLogs().operation = []
}

// 记录操作日志
const addOperationLog = (message, level = 'INFO') => {
  const timestamp = new Date().toLocaleString('zh-CN')
  const currentModeLogs = getCurrentModeLogs()
  currentModeLogs.operation.push({
    timestamp,
    level,
    message,
    logger: 'operation'
  })
  if (currentModeLogs.operation.length > 1000) {
    currentModeLogs.operation.shift()
  }
}

// 监听各种操作，记录到操作日志
const handleModelChanged = () => {
  addOperationLog('检测模型已切换', 'INFO')
}

const handleVideoChanged = () => {
  addOperationLog('视频源已切换', 'INFO')
}

const handleClassesChanged = () => {
  addOperationLog('检测类别配置已更新', 'INFO')
}

const handleDisplayChanged = () => {
  // 显示配置更新后，通知VideoPanel重新加载配置
  if (videoPanelRef.value && videoPanelRef.value.reloadDisplayConfig) {
    videoPanelRef.value.reloadDisplayConfig()
  }
  addOperationLog('显示配置已更新', 'INFO')
}

const handleAlarmChanged = () => {
  addOperationLog('报警配置已更新', 'INFO')
}

const goToMonitor = () => {
  router.push('/monitor')
}

// 检查登录状态
const checkLoginStatus = async () => {
  try {
    const res = await axios.get('/api/login/status')
    loginEnabled.value = res.data.login_enabled
  } catch (error) {
    console.error('检查登录状态失败:', error)
  }
}

// 登出
const handleLogout = async () => {
  try {
    const res = await axios.post('/api/logout')
    if (res.data.success) {
      // 跳转到登录页
      router.push('/login')
    }
  } catch (error) {
    console.error('登出失败:', error)
    // 即使失败也跳转到登录页
    router.push('/login')
  }
}


</script>

<style scoped>
.app-container {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.app-header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 20px 30px;
  height: auto !important;
}

.header-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
}

.header-content h1 {
  margin: 0;
  font-size: 24px;
  font-weight: 600;
}

.status-bar {
  display: flex;
  gap: 15px;
  align-items: center;
}

.video-row {
  margin-bottom: 20px;
}

/* 下方：区域管理、日志、设置 */
.bottom-nav-bar {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  overflow: hidden;
}

.top-tabs {
  margin: 0;
}

.top-tabs :deep(.el-tabs__header) {
  margin: 0;
  background: #f5f7fa;
  padding: 0 20px;
}

.top-tabs :deep(.el-tabs__item) {
  height: 50px;
  line-height: 50px;
  font-size: 15px;
  font-weight: 500;
  padding: 0 30px;
  border-bottom: 3px solid transparent;
  transition: all 0.3s;
}

.top-tabs :deep(.el-tabs__item.is-active) {
  color: #409eff;
  border-bottom-color: #409eff;
  background: white;
}

.top-tabs :deep(.el-tabs__item:hover) {
  color: #409eff;
}

.tab-content {
  padding: 20px;
  background: white;
  min-height: 400px;
  max-height: 600px;
  overflow-y: auto;
}

.log-tabs {
  border: none;
}

.log-tabs :deep(.el-tabs__header) {
  margin-bottom: 15px;
}

.main-content {
  padding: 20px;
  background: #f5f7fa;
}

.info-card {
  height: 100%;
  border-radius: 8px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

.info-card :deep(.el-card__body) {
  padding: 0;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-radius: 8px 8px 0 0;
}

.card-title {
  font-size: 16px;
  font-weight: 600;
}

.info-tabs {
  border: none;
}

.info-tabs :deep(.el-tabs__header) {
  margin: 0;
  background: #f5f7fa;
  padding: 0 15px;
}

.info-tabs :deep(.el-tabs__content) {
  padding: 15px;
  min-height: 400px;
  max-height: 600px;
  overflow-y: auto;
}

.log-container {
  padding: 10px;
}

.log-header {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  margin-bottom: 10px;
  padding-bottom: 10px;
  border-bottom: 1px solid #e4e7ed;
}

.alarm-tab-wrap {
  padding: 0;
}

.alarm-header {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  margin-bottom: 10px;
  padding-bottom: 10px;
  border-bottom: 1px solid #e4e7ed;
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

.flash-confirm-card {
  min-width: 420px;
  max-width: 70vw;
  background: #ffffff;
  border: 2px solid #f56c6c;
  border-radius: 10px;
  box-shadow: 0 8px 26px rgba(0, 0, 0, 0.3);
  padding: 18px 22px;
  text-align: center;
  pointer-events: auto;
}

.flash-confirm-title {
  font-size: 20px;
  font-weight: 700;
  color: #c45656;
  margin-bottom: 8px;
}

.flash-confirm-text {
  color: #333;
  font-size: 16px;
  line-height: 1.6;
  margin-bottom: 16px;
}

.flash-confirm-actions {
  display: flex;
  justify-content: center;
  gap: 10px;
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

<style>
.alarm-confirm-dialog {
  width: 560px !important;
}

.alarm-confirm-dialog .el-message-box__title {
  font-size: 22px;
}

.alarm-confirm-dialog .el-message-box__message {
  font-size: 18px;
  line-height: 1.7;
}

.alarm-confirm-dialog .el-button {
  min-width: 170px;
  font-size: 16px;
}
</style>

