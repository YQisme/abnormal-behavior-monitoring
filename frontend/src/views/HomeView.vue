<template>
  <div class="home-page">
    <div class="hero-card">
      <h1>智能监测系统</h1>
      <p>请选择需要进入的功能模块</p>
      <div class="nav-actions">
        <el-button type="primary" size="large" @click="go('/zone-alarm')">区域报警</el-button>
        <el-button type="warning" size="large" @click="go('/leave-monitor')">离岗监测</el-button>
        <el-button type="success" size="large" @click="go('/drowsy-monitor')">瞌睡监测</el-button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { useRouter } from 'vue-router'

const router = useRouter()

const go = (path) => {
  router.push(path)
}
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
