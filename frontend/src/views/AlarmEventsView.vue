<template>
  <div class="alarm-events-page">
    <el-card shadow="hover" class="events-card">
      <template #header>
        <div class="card-header">
          <span>报警事件</span>
          <div class="header-actions">
            <el-tag type="info">共 {{ items.length }} 条</el-tag>
            <el-button type="primary" size="small" @click="fetchEvents" :loading="loading">刷新</el-button>
          </div>
        </div>
      </template>

      <div class="toolbar">
        <el-radio-group v-model="activeType" size="small" @change="fetchEvents">
          <el-radio-button label="all">全部</el-radio-button>
          <el-radio-button label="image">图片</el-radio-button>
          <el-radio-button label="video">视频</el-radio-button>
        </el-radio-group>
      </div>

      <el-empty v-if="!loading && items.length === 0" description="暂无报警事件文件" />

      <div v-else class="events-grid" v-loading="loading">
        <el-card v-for="item in items" :key="item.relative_path" class="media-item" shadow="never">
          <template #header>
            <div class="media-header">
              <el-tag size="small" :type="item.media_type === 'image' ? 'success' : 'warning'">
                {{ item.media_type === 'image' ? '图片' : '视频' }}
              </el-tag>
              <span class="event-type">{{ item.event_type }}</span>
            </div>
          </template>

          <div class="media-body">
            <el-image
              v-if="item.media_type === 'image'"
              :src="item.media_url"
              fit="cover"
              class="image-preview"
              :preview-src-list="[item.media_url]"
              preview-teleported
            />
            <video v-else :src="item.media_url" controls class="video-preview" />
          </div>

          <div class="meta">
            <div class="name" :title="item.name">{{ item.name }}</div>
            <div class="time">{{ formatTime(item.modified_at) }}</div>
          </div>
        </el-card>
      </div>
    </el-card>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'
import { ElMessage } from 'element-plus'

const loading = ref(false)
const activeType = ref('all')
const items = ref([])

const fetchEvents = async () => {
  loading.value = true
  try {
    const res = await axios.get('/api/alarm-events', {
      params: {
        type: activeType.value,
        limit: 300
      }
    })
    if (res.data?.success) {
      items.value = res.data.items || []
    } else {
      ElMessage.warning(res.data?.message || '获取报警事件失败')
    }
  } catch (error) {
    console.error('获取报警事件失败:', error)
    ElMessage.error(error.response?.data?.message || '获取报警事件失败')
  } finally {
    loading.value = false
  }
}

const formatTime = (timestamp) => {
  if (!timestamp) return '-'
  return new Date(timestamp * 1000).toLocaleString('zh-CN')
}

onMounted(() => {
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
  margin-bottom: 16px;
}

.events-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 16px;
}

.media-item {
  border: 1px solid #ebeef5;
}

.media-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}

.event-type {
  color: #909399;
  font-size: 12px;
}

.media-body {
  width: 100%;
  aspect-ratio: 16 / 9;
  background: #f5f7fa;
  overflow: hidden;
  border-radius: 6px;
}

.image-preview,
.video-preview {
  width: 100%;
  height: 100%;
}

.meta {
  margin-top: 10px;
}

.name {
  font-size: 13px;
  color: #303133;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.time {
  margin-top: 6px;
  color: #909399;
  font-size: 12px;
}
</style>
