<template>
  <div class="alarm-events-page">
    <el-card shadow="hover" class="events-card">
      <template #header>
        <div class="card-header">
          <span>报警事件</span>
          <div class="header-actions">
            <el-tag type="info">事件 {{ totalEvents }} 条</el-tag>
            <el-button size="small" @click="handleExport('json')">导出文档(JSON)</el-button>
            <el-button size="small" @click="handleExport('csv')">导出文档(CSV)</el-button>
            <el-button type="primary" size="small" @click="fetchEvents" :loading="loading">刷新</el-button>
          </div>
        </div>
      </template>

      <div class="summary-row" v-if="storage">
        <el-tag type="success">事件占用：{{ formatBytes(storage.events_used_bytes) }}</el-tag>
        <el-tag type="warning">磁盘剩余：{{ formatBytes(storage.disk_free_bytes) }}</el-tag>
        <el-tag type="info">磁盘总量：{{ formatBytes(storage.disk_total_bytes) }}</el-tag>
      </div>

      <div class="toolbar">
        <div class="toolbar-row">
          <el-radio-group v-model="selectedEventType" size="small" @change="fetchEvents">
            <el-radio-button label="all">全部类型</el-radio-button>
            <el-radio-button label="zone">区域报警</el-radio-button>
            <el-radio-button label="offpost">离岗报警</el-radio-button>
            <el-radio-button label="drowsy">瞌睡报警</el-radio-button>
            <el-radio-button label="unknown">其他</el-radio-button>
          </el-radio-group>
          <el-radio-group v-model="viewMode" size="small">
            <el-radio-button label="timeline">时间线</el-radio-button>
            <el-radio-button label="grid">平铺</el-radio-button>
            <el-radio-button label="document">文档视图</el-radio-button>
          </el-radio-group>
        </div>
        <div class="toolbar-row">
          <el-date-picker
            v-model="timeRange"
            type="datetimerange"
            range-separator="至"
            start-placeholder="开始时间"
            end-placeholder="结束时间"
            format="YYYY-MM-DD HH:mm:ss"
            value-format="x"
            @change="fetchEvents"
          />
          <el-button @click="clearTimeRange">清空时间</el-button>
          <el-button
            v-if="viewMode === 'grid'"
            type="danger"
            :disabled="selectedMediaPaths.length === 0"
            @click="handleBatchDelete"
          >
            批量删除已选（{{ selectedMediaPaths.length }}）
          </el-button>
        </div>
      </div>

      <el-empty v-if="!loading && filteredEvents.length === 0" description="暂无报警事件数据" />

      <div v-else-if="viewMode === 'timeline'" class="timeline-wrap" v-loading="loading">
        <el-timeline>
          <el-timeline-item
            v-for="event in filteredEvents"
            :key="event.event_id"
            :timestamp="formatTime(event.occurred_at)"
            placement="top"
          >
            <el-card class="event-card" :class="{ 'event-card-target': isTargetEvent(event) }" shadow="never" :data-event-id="event.event_id">
              <template #header>
                <div class="event-header">
                  <div class="event-tags">
                    <el-tag size="small">{{ eventTypeLabel(event.event_type) }}</el-tag>
                    <el-tag size="small" type="info">占用 {{ formatBytes(event.total_size) }}</el-tag>
                  </div>
                  <div class="event-actions">
                    <el-button
                      v-if="event.image"
                      size="small"
                      type="danger"
                      link
                      @click="handleDeleteSingle(event.image)"
                    >
                      删除图片
                    </el-button>
                    <el-button
                      v-if="event.video"
                      size="small"
                      type="danger"
                      link
                      @click="handleDeleteSingle(event.video)"
                    >
                      删除视频
                    </el-button>
                    <el-button size="small" type="danger" @click="handleDeleteEvent(event)">删除整条事件</el-button>
                  </div>
                </div>
              </template>

              <div class="event-content">
                <div class="media-column">
                  <div class="media-block" v-if="event.image">
                    <div class="media-title">图片</div>
                    <el-image
                      :src="event.image.media_url"
                      fit="cover"
                      class="image-preview"
                      :preview-src-list="[event.image.media_url]"
                      lazy
                      preview-teleported
                    />
                  </div>
                  <div class="media-block" v-if="event.video">
                    <div class="media-title">视频</div>
                    <video :src="event.video.media_url" controls preload="none" class="video-preview" />
                  </div>
                </div>
                <div class="doc-column">
                  <div class="doc-title">事件文档</div>
                  <el-descriptions :column="1" border size="small">
                    <el-descriptions-item label="事件ID">{{ event.document?.event_id || event.event_id }}</el-descriptions-item>
                    <el-descriptions-item label="事件类型">{{ eventTypeLabel(event.document?.event_type || event.event_type) }}</el-descriptions-item>
                    <el-descriptions-item label="发生时间">{{ formatTime(event.document?.occurred_at || event.occurred_at) }}</el-descriptions-item>
                    <el-descriptions-item label="目标">{{ event.document?.object_name || '-' }}</el-descriptions-item>
                    <el-descriptions-item label="区域">{{ event.document?.zone_name || '-' }}</el-descriptions-item>
                    <el-descriptions-item label="跟踪ID">{{ event.document?.track_id || '-' }}</el-descriptions-item>
                    <el-descriptions-item label="说明">{{ event.document?.description || '无' }}</el-descriptions-item>
                  </el-descriptions>
                </div>
              </div>
            </el-card>
          </el-timeline-item>
        </el-timeline>
      </div>

      <div v-else-if="viewMode === 'grid'" class="grid-wrap" v-loading="loading">
        <el-row :gutter="16">
          <el-col
            v-for="event in filteredEvents"
            :key="event.event_id"
            :xs="24"
            :sm="12"
            :md="12"
            :lg="8"
          >
            <el-card class="event-card grid-card" :class="{ 'event-card-target': isTargetEvent(event) }" shadow="never" :data-event-id="event.event_id">
              <template #header>
                <div class="event-header">
                  <div class="event-tags">
                    <el-tag size="small">{{ eventTypeLabel(event.event_type) }}</el-tag>
                    <el-tag size="small" type="info">{{ formatTime(event.occurred_at) }}</el-tag>
                  </div>
                  <el-button size="small" type="danger" @click="handleDeleteEvent(event)">删事件</el-button>
                </div>
              </template>
              <div class="grid-media">
                <div v-if="event.image" class="media-unit">
                  <div class="media-row">
                    <el-checkbox :model-value="isSelected(event.image.relative_path)" @change="toggleSelected(event.image.relative_path)" />
                    <span>图片</span>
                    <el-button size="small" type="danger" link @click="handleDeleteSingle(event.image)">删除</el-button>
                  </div>
                  <el-image
                    :src="event.image.media_url"
                    fit="cover"
                    class="image-preview"
                    :preview-src-list="[event.image.media_url]"
                    lazy
                    preview-teleported
                  />
                </div>
                <div v-if="event.video" class="media-unit">
                  <div class="media-row">
                    <el-checkbox :model-value="isSelected(event.video.relative_path)" @change="toggleSelected(event.video.relative_path)" />
                    <span>视频</span>
                    <el-button size="small" type="danger" link @click="handleDeleteSingle(event.video)">删除</el-button>
                  </div>
                  <video :src="event.video.media_url" controls preload="none" class="video-preview" />
                </div>
              </div>
            </el-card>
          </el-col>
        </el-row>
      </div>

      <div v-else class="document-wrap" v-loading="loading">
        <el-table :data="documentRows" border stripe>
          <el-table-column prop="event_id" label="事件ID" min-width="200" show-overflow-tooltip />
          <el-table-column prop="event_type_label" label="事件类型" width="110" />
          <el-table-column prop="occurred_time" label="发生时间" width="180" />
          <el-table-column prop="object_name" label="目标" min-width="120" show-overflow-tooltip />
          <el-table-column prop="zone_name" label="区域" min-width="120" show-overflow-tooltip />
          <el-table-column prop="track_id" label="跟踪ID" width="100" />
          <el-table-column prop="description" label="说明" min-width="240" show-overflow-tooltip />
          <el-table-column prop="total_size_text" label="占用大小" width="110" />
        </el-table>
      </div>
    </el-card>
  </div>
</template>

<script setup>
import { computed, ref, onMounted, nextTick } from 'vue'
import axios from 'axios'
import { ElMessage, ElMessageBox } from 'element-plus'
import { useRoute } from 'vue-router'

const loading = ref(false)
const selectedEventType = ref('all')
const groupedTimeline = ref({})
const storage = ref(null)
const viewMode = ref('timeline')
const timeRange = ref([])
const selectedMediaPaths = ref([])
const targetEventId = ref('')
const route = useRoute()

const fetchEvents = async () => {
  loading.value = true
  try {
    const startTime = Array.isArray(timeRange.value) && timeRange.value[0] ? Number(timeRange.value[0]) / 1000 : undefined
    const endTime = Array.isArray(timeRange.value) && timeRange.value[1] ? Number(timeRange.value[1]) / 1000 : undefined
    const res = await axios.get('/api/alarm-events', {
      params: {
        // 控制单次媒体数量，避免瞬时加载过多图片/视频导致内存压力
        limit: 80,
        event_type: selectedEventType.value,
        start_time: startTime,
        end_time: endTime
      }
    })
    if (res.data?.success) {
      groupedTimeline.value = res.data.grouped_timeline || {}
      storage.value = res.data.storage || null
      selectedMediaPaths.value = []
    } else {
      ElMessage.warning(res.data?.message || '获取报警事件失败')
    }
  } catch (error) {
    console.error('获取报警事件失败:', error)
    ElMessage.error(error.response?.data?.message || '获取报警事件失败')
  } finally {
    loading.value = false
    await nextTick()
    scrollToTargetEvent()
  }
}

const flatEvents = computed(() => {
  return Object.values(groupedTimeline.value || {}).flat().sort((a, b) => (b.occurred_at || 0) - (a.occurred_at || 0))
})

const filteredEvents = computed(() => {
  return flatEvents.value
})

const totalEvents = computed(() => flatEvents.value.length)

const documentRows = computed(() => {
  return filteredEvents.value.map((event) => {
    const doc = event.document || {}
    const occurredAt = doc.occurred_at || event.occurred_at
    return {
      event_id: doc.event_id || event.event_id,
      event_type_label: eventTypeLabel(doc.event_type || event.event_type),
      occurred_time: formatTime(occurredAt),
      object_name: doc.object_name || '-',
      zone_name: doc.zone_name || '-',
      track_id: doc.track_id || '-',
      description: doc.description || '无',
      total_size_text: formatBytes(event.total_size)
    }
  })
})

const formatTime = (timestamp) => {
  if (!timestamp) return '-'
  return new Date(timestamp * 1000).toLocaleString('zh-CN')
}

const formatBytes = (bytes) => {
  const value = Number(bytes || 0)
  if (value <= 0) return '0 B'
  const units = ['B', 'KB', 'MB', 'GB', 'TB']
  let size = value
  let idx = 0
  while (size >= 1024 && idx < units.length - 1) {
    size /= 1024
    idx += 1
  }
  return `${size.toFixed(idx === 0 ? 0 : 2)} ${units[idx]}`
}

const eventTypeLabel = (type) => {
  if (type === 'zone') return '区域报警'
  if (type === 'offpost') return '离岗报警'
  if (type === 'drowsy') return '瞌睡报警'
  return '其他'
}

const isTargetEvent = (event) => {
  if (!targetEventId.value) return false
  return (event?.event_id || '') === targetEventId.value
}

const scrollToTargetEvent = () => {
  if (!targetEventId.value) return
  const targetEl = document.querySelector(`[data-event-id="${targetEventId.value}"]`)
  if (!targetEl) return
  targetEl.scrollIntoView({ behavior: 'smooth', block: 'center' })
}

const handleDeleteSingle = async (media) => {
  try {
    await ElMessageBox.confirm(`确认删除该${media.media_type === 'image' ? '图片' : '视频'}？`, '删除确认', {
      type: 'warning'
    })
    const res = await axios.delete('/api/alarm-events', {
      data: {
        relative_path: media.relative_path,
        delete_scope: 'single'
      }
    })
    if (res.data?.success) {
      ElMessage.success('删除成功')
      fetchEvents()
    } else {
      ElMessage.warning(res.data?.message || '删除失败')
    }
  } catch (error) {
    if (error !== 'cancel') {
      ElMessage.error(error.response?.data?.message || '删除失败')
    }
  }
}

const handleDeleteEvent = async (event) => {
  try {
    await ElMessageBox.confirm('确认删除该事件下的图片和视频？', '删除确认', {
      type: 'warning'
    })
    const relativePath = event.image?.relative_path || event.video?.relative_path || ''
    const res = await axios.delete('/api/alarm-events', {
      data: {
        event_id: event.event_id,
        relative_path: relativePath,
        delete_scope: 'event'
      }
    })
    if (res.data?.success) {
      ElMessage.success('事件已删除')
      fetchEvents()
    } else {
      ElMessage.warning(res.data?.message || '删除失败')
    }
  } catch (error) {
    if (error !== 'cancel') {
      ElMessage.error(error.response?.data?.message || '删除失败')
    }
  }
}

const handleExport = async (format) => {
  try {
    const startTime = Array.isArray(timeRange.value) && timeRange.value[0] ? Number(timeRange.value[0]) / 1000 : undefined
    const endTime = Array.isArray(timeRange.value) && timeRange.value[1] ? Number(timeRange.value[1]) / 1000 : undefined
    const res = await axios.get('/api/alarm-events/export', {
      params: {
        format,
        event_type: selectedEventType.value,
        start_time: startTime,
        end_time: endTime
      },
      responseType: 'blob'
    })
    const blob = new Blob([res.data], {
      type: format === 'csv' ? 'text/csv;charset=utf-8' : 'application/json;charset=utf-8'
    })
    const url = window.URL.createObjectURL(blob)
    const link = document.createElement('a')
    const now = new Date().toISOString().replace(/[:.]/g, '-')
    link.href = url
    link.download = `alarm_events_documents_${selectedEventType.value}_${now}.${format}`
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
    window.URL.revokeObjectURL(url)
    ElMessage.success('导出成功')
  } catch (error) {
    console.error('导出报警事件文档失败:', error)
    ElMessage.error(error.response?.data?.message || '导出失败')
  }
}

const clearTimeRange = () => {
  timeRange.value = []
  fetchEvents()
}

const isSelected = (relativePath) => selectedMediaPaths.value.includes(relativePath)

const toggleSelected = (relativePath) => {
  const idx = selectedMediaPaths.value.indexOf(relativePath)
  if (idx >= 0) {
    selectedMediaPaths.value.splice(idx, 1)
  } else {
    selectedMediaPaths.value.push(relativePath)
  }
}

const handleBatchDelete = async () => {
  if (selectedMediaPaths.value.length === 0) return
  try {
    await ElMessageBox.confirm(`确认删除已选择的 ${selectedMediaPaths.value.length} 个文件？`, '批量删除确认', {
      type: 'warning'
    })
    const res = await axios.delete('/api/alarm-events', {
      data: {
        relative_paths: selectedMediaPaths.value,
        delete_scope: 'single'
      }
    })
    if (res.data?.success) {
      ElMessage.success(`已删除 ${res.data.deleted_count || 0} 个文件`)
      fetchEvents()
    } else {
      ElMessage.warning(res.data?.message || '批量删除失败')
    }
  } catch (error) {
    if (error !== 'cancel') {
      ElMessage.error(error.response?.data?.message || '批量删除失败')
    }
  }
}

onMounted(() => {
  const queryEventType = String(route.query.event_type || '').trim()
  if (queryEventType && ['all', 'zone', 'offpost', 'drowsy', 'unknown'].includes(queryEventType)) {
    selectedEventType.value = queryEventType
  }
  const queryView = String(route.query.view || '').trim()
  if (queryView && ['timeline', 'grid', 'document'].includes(queryView)) {
    viewMode.value = queryView
  }
  targetEventId.value = String(route.query.eventId || '').trim()
  fetchEvents()
})
</script>

<style scoped>
.alarm-events-page {
  padding: 20px;
}

.events-card {
  border-radius: 10px;
}

.card-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 10px;
}

.toolbar {
  margin: 16px 0;
}

.toolbar-row {
  display: flex;
  gap: 10px;
  align-items: center;
  flex-wrap: wrap;
  margin-bottom: 10px;
}

.summary-row {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.timeline-wrap {
  margin-top: 10px;
}

.event-card {
  border: 1px solid #ebeef5;
}

.event-card-target {
  border-color: #409eff;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.25);
}

.event-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 10px;
}

.event-tags,
.event-actions {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.event-content {
  display: grid;
  grid-template-columns: 1.4fr 1fr;
  gap: 16px;
}

.media-column {
  display: grid;
  gap: 12px;
}

.media-block {
  border: 1px solid #ebeef5;
  border-radius: 8px;
  padding: 10px;
}

.media-title,
.doc-title {
  font-size: 13px;
  color: #606266;
  margin-bottom: 8px;
}

.image-preview,
.video-preview {
  width: 100%;
  aspect-ratio: 16 / 9;
  background: #f5f7fa;
  border-radius: 6px;
}

.doc-column {
  min-width: 0;
}

.grid-wrap {
  margin-top: 10px;
}

.document-wrap {
  margin-top: 10px;
}

.grid-card {
  margin-bottom: 16px;
}

.grid-media {
  display: grid;
  gap: 10px;
}

.media-unit {
  border: 1px solid #ebeef5;
  border-radius: 8px;
  padding: 8px;
}

.media-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

@media (max-width: 1100px) {
  .event-content {
    grid-template-columns: 1fr;
  }
}
</style>
