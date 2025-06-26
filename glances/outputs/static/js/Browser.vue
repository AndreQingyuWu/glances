<template>
  <!-- 移除外层容器 -->
  <div class="app-wrapper">
    <!-- 标题区域 -->
    <header class="app-header">
      <h1>西三环-人工智能服务器</h1>
      <p>实时监控与管理 AI 服务器集群状态</p>
    </header>
    
    <!-- 加载状态 -->
    <div v-if="!serversListLoaded" class="loading-container">
      <div class="loader">
        <div class="spinner"></div>
        <div class="loader-text">Glances Central Browser 正在加载中...</div>
      </div>
    </div>
    
    <!-- 主内容区域 - 添加滚动容器 -->
    <main v-else class="scrollable-container">
      <div class="content-container">
        <!-- 无服务器状态 -->
        <div v-if="servers.length === 0" class="empty-state">
          <div class="empty-icon">
            <i class="fas fa-server"></i>
          </div>
          <p class="title">没有可用的 Glances 服务器</p>
          <p>可以在 glances.conf 文件中配置 Glances 服务器。</p>
          <p>系统可以在本地局域网中自动检测 Glances 服务器。</p>
        </div>
        
        <!-- 有服务器时的显示 -->
        <div v-else>
          <!-- 服务器统计信息 - 修改为单行布局 -->
          <div class="server-stats">
            <div class="stats-title">
              <span v-if="servers.length === 1">可用服务器: <span class="stats-count">1 台</span></span>
              <span v-if="servers.length > 1">可用服务器: <span class="stats-count">{{ servers.length }} 台</span></span>
            </div>
            <button class="refresh-btn" @click="updateServersList">
              <i class="fas fa-sync-alt"></i> 刷新列表
            </button>
          </div>
          
          <!-- 服务器表格 - 移动端使用卡片布局 -->
          <div class="table-container">
            <!-- 桌面端表格视图 -->
            <table class="server-table desktop-view">
              <thead>
                <tr>
                  <th>服务器名称</th>
                  <th>IP 地址</th>
                  <th>状态</th>
                  <th>协议</th>
                  <th v-if="servers.length" v-for="(column, columnId) in servers[0].columns" :key="columnId">
                    {{ column.replace(/_/g, ' ') }}
                  </th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(server, serverId) in servers" :key="serverId" @click="goToGlances(server)">
                  <td>
                    <div class="server-name">
                      <i class="fas fa-server"></i>
                      {{ server.alias ? server.alias : server.name }}
                    </div>
                  </td>
                  <td>
                    {{ server.ip }}
                    <button class="details-btn" @click.stop="goToGlances(server)">
                      点击查看详情 <i class="fas fa-arrow-right"></i>
                    </button>
                  </td>
                  <td>
                    <div class="status-cell">
                      <span class="status-indicator" :class="getStatusClass(server)"></span>
                      {{ server.status }}
                    </div>
                  </td>
                  <td>
                    <span class="protocol-tag" :class="getProtocolClass(server)">
                      {{ server.protocol }}
                    </span>
                  </td>
                  <td v-if="servers.length" v-for="(column, columnId) in server.columns" :key="columnId" 
                      :class="getDecoration(server, column)">
                    {{ formatNumber(server[column]) }}
                  </td>
                </tr>
              </tbody>
            </table>
            
            <!-- 移动端卡片视图 -->
            <div class="mobile-view">
              <div v-for="(server, serverId) in servers" :key="serverId" class="server-card" @click="goToGlances(server)">
                <div class="card-header">
                  <div class="server-name">
                    <i class="fas fa-server"></i>
                    <h3>{{ server.alias ? server.alias : server.name }}</h3>
                  </div>
                  <div class="status-cell">
                    <span class="status-indicator" :class="getStatusClass(server)"></span>
                    <span>{{ server.status }}</span>
                  </div>
                </div>
                
                <div class="card-body">
                  <div class="info-row">
                    <span class="info-label">IP 地址:</span>
                    <span class="info-value">{{ server.ip }}</span>
                  </div>
                  
                  <div class="info-row">
                    <span class="info-label">协议:</span>
                    <span class="protocol-tag" :class="getProtocolClass(server)">
                      {{ server.protocol }}
                    </span>
                  </div>
                  
                  <div v-for="(column, columnId) in server.columns" :key="columnId" class="info-row">
                    <span class="info-label">{{ column.replace(/_/g, ' ') }}:</span>
                    <span class="info-value" :class="getDecoration(server, column)">
                      {{ formatNumber(server[column]) }}
                    </span>
                  </div>
                </div>
                
                <button class="details-btn mobile-btn" @click.stop="goToGlances(server)">
                  点击查看详情 <i class="fas fa-arrow-right"></i>
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script>
export default {
  data() {
    return {
      servers: undefined,
    };
  },
  computed: {
    serversListLoaded() {
      return this.servers !== undefined;
    },
  },
  created() {
    this.updateServersList();
  },
  mounted() {
    const GLANCES = window.__GLANCES__ || {};
    const refreshTime = isFinite(GLANCES['refresh-time'])
      ? parseInt(GLANCES['refresh-time'], 10)
      : undefined;
    this.interval = setInterval(this.updateServersList, refreshTime * 1000)
  },
  methods: {
    updateServersList() {
      fetch('api/4/serverslist', { method: 'GET' })
        .then((response) => response.json())
        .then((response) => (this.servers = response));
    },
    formatNumber(value) {
      if (typeof value === "number" && !isNaN(value)) {
        return value.toFixed(1);
      }
      return value;
    },
    goToGlances(server) {
      if (server.protocol === 'rpc') {
        alert("You just click on a Glances RPC server.\nPlease open a terminal and enter the following command line:\n\nglances -c ${server.ip}:${server.port}")
      } else {
        let newUri = server.uri;
        if (newUri.includes('192.168.4.5')) {
          newUri = newUri.replace('192.168.4.5', '36.133.163.46');
        } else if (newUri.includes('192.168.4.8')) {
          newUri = newUri.replace('192.168.4.8', '36.134.146.191');
        }
        window.location.href = newUri;
      }
    },
    getDecoration(server, column) {
      if (server[column + '_decoration'] === undefined) {
        return;
      }
      const decoration = server[column + '_decoration'].replace('_LOG', '').toLowerCase();
      return `decoration-${decoration}`;
    },
    getStatusClass(server) {
      if (server.status.includes('运行') || server.status.toLowerCase().includes('active')) 
        return 'status-active';
      if (server.status.includes('警告') || server.status.toLowerCase().includes('warning')) 
        return 'status-warning';
      return 'status-error';
    },
    getProtocolClass(server) {
      return server.protocol === 'http' ? 'protocol-http' : 'protocol-rpc';
    }
  },
  destroyed() {
    clearInterval(this.interval)
  }
};
</script>

<style scoped>
/* === 关键修改：确保页面可以滚动 === */
html, body {
  height: 100%;
  margin: 0;
  padding: 0;
  overflow: auto !important;
}

.app-wrapper {
  display: flex;
  flex-direction: column;
  height: 100vh;
}

.scrollable-container {
  overflow-y: auto;
  flex: 1;
  padding: 0 10px 30px; /* 添加底部内边距确保按钮完全显示 */
  margin-top: 0px;
}

/* === 结束关键修改 === */

/* 基础样式 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: 'Segoe UI', 'Microsoft YaHei', sans-serif;
}

:root {
  --primary: #2563eb;
  --primary-light: #3b82f6;
  --primary-dark: #1d4ed8;
  --secondary: #374151;
  --success: #25c892;
  --warning: #f59e0b;
  --danger: #ef4444;
  --light: #f9fafb;
  --dark: #111827;
  --gray: #6b7280;
  --border: #e5e7eb;
  --card-bg: #ffffff;
  --bg-gradient: linear-gradient(135deg, #1e3a8a 0%, #2563eb 100%);
  --shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
  --radius: 12px;
}

/* 标题样式 */
.app-header {
  background: var(--bg-gradient);
  color: white;
  padding: 20px 10px 0px;
  text-align: center;
  position: sticky;
  top: 0;
  z-index: 100;
  box-shadow: var(--shadow);
  text-align: center;
  display: flex; /* 新增：使用flex布局 */
  flex-direction: column; /* 垂直排列 */
  align-items: center;
}

.app-header h1 {
  font-weight: 700;
  font-size: clamp(1.5rem, 4vw, 2.2rem);
  margin: 10px auto 8px;
  letter-spacing: 0.5px;
}

.app-header p {
  font-weight: 300;
  opacity: 0.9;
  font-size: clamp(0.9rem, 2vw, 1.1rem);
  max-width: 600px;
  margin: 0 auto 8px;
}

/* 加载状态 */
.loading-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 200px;
  background: var(--card-bg);
  border-radius: var(--radius);
  box-shadow: var(--shadow);
  margin: 20px;
}

.loader {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
}

.spinner {
  width: 50px;
  height: 50px;
  border: 4px solid rgba(37, 99, 235, 0.2);
  border-top: 4px solid var(--primary);
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 15px;
}

.loader-text {
  font-size: 1.1rem;
  font-weight: 500;
  color: var(--secondary);
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

/* 空状态 */
.empty-state {
  background: var(--card-bg);
  border-radius: var(--radius);
  padding: 30px 20px;
  text-align: center;
  box-shadow: var(--shadow);
  margin: 20px;
}

.empty-icon {
  font-size: 3rem;
  color: var(--gray);
  margin-bottom: 15px;
  opacity: 0.7;
}

.empty-state .title {
  font-size: 1.5rem;
  font-weight: 600;
  color: var(--secondary);
  margin-bottom: 10px;
}

.empty-state p {
  margin: 8px 0;
  color: var(--gray);
  font-size: 0.95rem;
  line-height: 1.6;
}

/* 服务器统计 - 关键修改：移动端单行显示 */
.server-stats {
  display: flex;
  flex-direction: column;
  gap: 10px;
  background: var(--card-bg);
  padding: 10px;
  border-radius: var(--radius);
  margin-bottom: 10px;
  margin-top: 10px;
  box-shadow: var(--shadow);
  border-left: 4px solid var(--primary);
}

@media (min-width: 576px) {
  .server-stats {
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
    padding: 15px 20px;
  }
}

/* 移动端单行显示 */
@media (max-width: 575px) {
  .server-stats {
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
    padding: 12px 12px;
    gap: 10px;
  }
  
  .stats-title {
    font-size: 0.9rem;
    white-space: nowrap;
    overflow: visible;
    text-overflow: ellipsis;
    flex-shrink: 1;
    padding-left: 10px;
  }
  
  .refresh-btn {
    border: white;
    font-size: 0.8rem;
    white-space: nowrap;
    flex-shrink: 0;
    align-items: center;
    justify-content: center;
    padding-right: 10px;
  }
}

.stats-title {
  font-size: 0.9rem;
  font-weight: 600;
  color: var(--secondary);
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  padding-left: 10px;
  flex-shrink: 1;
}

.stats-count {
  background: var(--primary);
  color: white;
  border-radius: 20px;
  font-weight: 700;
  margin-left: 1px;
  font-size: 0.9rem;
}

.refresh-btn {
  background: var(--primary);
  border: white;
  color: white;
  border: none;
  padding-right: 10px;
  border-radius: 8px;
  font-weight: 500;
  cursor: pointer;
  display: flex;
  align-items: center;
  transition: all 0.2s ease;
  box-shadow: 0 2px 4px rgba(37, 99, 235, 0.2);
  font-size: 0.8rem;
  width: 100%;
  justify-content: center;
}

@media (min-width: 576px) {
  .refresh-btn {
    width: auto;
  }
}

.refresh-btn:hover {
  background: var(--primary-dark);
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(37, 99, 235, 0.3);
}

.refresh-btn i {
  margin-right: 8px;
}

/* 表格容器 */
.table-container {
  background: var(--card-bg);
  border-radius: var(--radius);
  box-shadow: var(--shadow);
  margin-bottom: 20px;
}

/* 桌面端表格视图 */
.server-table.desktop-view {
  width: 100%;
  border-collapse: collapse;
  display: none;
}

@media (min-width: 992px) {
  .server-table.desktop-view {
    display: table;
  }
}

.server-table.desktop-view thead {
  background-color: #f8fafc;
}

.server-table.desktop-view th {
  text-align: left;
  padding: 15px;
  font-weight: 600;
  color: var(--secondary);
  font-size: 0.9rem;
  border-bottom: 2px solid var(--border);
}

.server-table.desktop-view tbody tr {
  border-bottom: 1px solid var(--border);
  transition: background 0.2s ease;
}

.server-table.desktop-view tbody tr:hover {
  background-color: #f9fafb;
  cursor: pointer;
}

.server-table.desktop-view td {
  padding: 12px 15px;
  color: var(--dark);
  font-size: 0.9rem;
  vertical-align: middle;
}

.server-name {
  font-weight: 500;
  color: var(--dark);
  display: flex;
  align-items: center;
}

.server-name i {
  margin-right: 8px;
  color: var(--primary);
  font-size: 1rem;
}

.status-cell {
  display: flex;
  align-items: center;
}

.status-indicator {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  margin-right: 8px;
}

.status-active {
  background-color: var(--success);
}

.status-warning {
  background-color: var(--warning);
}

.status-error {
  background-color: var(--danger);
}

.protocol-tag {
  display: inline-block;
  padding: 4px 10px;
  border-radius: 20px;
  font-size: 0.8rem;
  font-weight: 500;
}

.protocol-http {
  background: rgba(16, 185, 129, 0.15);
  color: var(--success);
}

.protocol-rpc {
  background: rgba(245, 158, 11, 0.15);
  color: var(--warning);
}

.details-btn {
  background: var(--primary);
  color: #25c892;
  border: none;
  padding: 6px 12px;
  border-radius: 6px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  margin-top: 6px;
  box-shadow: 0 2px 4px rgba(37, 99, 235, 0.2);
  font-size: 0.85rem;
}

.details-btn:hover {
  background: var(--primary-dark);
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(37, 99, 235, 0.3);
}

.details-btn i {
  margin-left: 5px;
  font-size: 0.7rem;
}

/* 移动端卡片视图 */
.mobile-view {
  display: block;
  padding: 10px;
}

@media (min-width: 992px) {
  .mobile-view {
    display: none;
  }
}

.server-card {
  background: var(--card-bg);
  border-radius: 10px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  margin-bottom: 15px;
  overflow: hidden;
  transition: transform 0.2s ease;
}

.server-card:active {
  transform: scale(0.98);
}

.card-header {
  padding: 15px;
  border-bottom: 1px solid var(--border);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.card-header .server-name {
  display: flex;
  align-items: center;
  flex: 1;
}

.card-header .server-name i {
  margin-right: 10px;
  color: var(--primary);
}

.card-header .server-name h3 {
  font-size: 1.1rem;
  margin: 0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.card-header .status-cell {
  display: flex;
  align-items: center;
  margin-left: 10px;
}

.card-body {
  padding: 15px;
}

.info-row {
  display: flex;
  justify-content: space-between;
  margin-bottom: 12px;
  font-size: 0.95rem;
}

.info-row:last-child {
  margin-bottom: 0;
}

.info-label {
  color: var(--gray);
  font-weight: 500;
  margin-right: 10px;
}

.info-value {
  color: var(--dark);
  text-align: right;
  word-break: break-all;
}

.mobile-btn {
  width: calc(100% - 30px);
  margin: 0 15px 15px;
  padding: 10px;
  font-size: 0.95rem;
}

/* 装饰样式 */
.decoration-normal {
  color: var(--success);
  font-weight: 500;
}

.decoration-warning {
  color: var(--warning);
  font-weight: 500;
}

.decoration-critical {
  color: var(--danger);
  font-weight: 600;
}

/* 动画效果 */
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

.fade-in {
  animation: fadeIn 0.4s ease forwards;
}
</style>