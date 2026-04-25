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
        <el-menu-item index="/zone-alarm">
          <el-icon><WarningFilled /></el-icon>
          <span>区域报警</span>
        </el-menu-item>
        <el-menu-item index="/leave-monitor">
          <el-icon><UserFilled /></el-icon>
          <span>离岗监测</span>
        </el-menu-item>
      </el-menu>
    </el-aside>
    <el-main class="main-view">
      <router-view />
    </el-main>
  </el-container>
</template>

<script setup>
import { computed } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()
const hideSidebar = computed(() => route.path === '/login' || route.path === '/')
const activePath = computed(() => route.path)
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

.menu .el-menu-item:hover,
.menu .el-menu-item.is-active {
  background: rgba(255, 255, 255, 0.16) !important;
  color: #fff !important;
}

.main-view {
  padding: 0 !important;
  background: #f5f7fa;
}
</style>

