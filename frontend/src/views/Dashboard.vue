<template>
  <section class="page">
    <header class="page-head">
      <div>
        <h2>运营概览</h2>
        <p class="page-desc">汇总各业务模块的关键指标，先看总量再看异常。</p>
      </div>
      <div class="page-actions">
        <button class="btn" type="button" :disabled="loading" @click="retry">{{ loading ? '正在刷新…' : '刷新数据' }}</button>
      </div>
    </header>

    <!-- 首次取数且没有缓存：不给样例数字，明确告知正在取数 -->
    <div v-if="loading && !hasData" class="state-panel">
      <p>正在读取运营概览…</p>
    </div>

    <!-- 取数失败且没有可展示的数据：说清楚失败原因并提供重试，不回落任何样例 -->
    <div v-else-if="errorMessage && !hasData" class="state-panel error-panel">
      <p class="error-text">运营概览取数失败：{{ errorMessage }}</p>
      <button class="btn primary" type="button" :disabled="loading" @click="retry">
        {{ loading ? '重试中…' : '重新获取' }}
      </button>
    </div>

    <!-- 接口正常但确实没有数据：同样不展示任何样例数字 -->
    <div v-else-if="!hasData" class="state-panel">
      <p class="empty-state-text">暂无运营概览数据，各业务模块当前均无记录</p>
      <button class="btn" type="button" :disabled="loading" @click="retry">重新检查</button>
    </div>

    <!-- 有数据（本次接口返回或上次成功缓存）时才渲染卡片与明细 -->
    <template v-else>
      <div v-if="errorMessage" class="alert-bar error-text">
        本次刷新失败（{{ errorMessage }}），以下为{{ cacheLabel }}，点击「重新获取」可重试。
      </div>
      <div v-else-if="showCacheNotice" class="alert-bar">
        接口取数中，当前展示的是{{ cacheLabel }}，稍后自动更新。
      </div>

      <div class="stat-row">
        <article v-for="card in cards" :key="card.label" class="stat-card">
          <span class="stat-label">{{ card.label }}</span>
          <strong class="stat-value">{{ card.value }}</strong>
        </article>
      </div>
      <table class="data-table">
        <thead>
          <tr><th>业务模块</th><th>今日新增</th><th>待处理</th><th>异常量</th></tr>
        </thead>
        <tbody>
          <tr v-for="row in moduleRows" :key="row.name">
            <td>{{ moduleLabels[row.name] ?? row.name }}</td>
            <td>{{ row.created }}</td>
            <td>{{ row.pending }}</td>
            <td>{{ row.abnormal }}</td>
          </tr>
        </tbody>
      </table>

      <footer class="page-foot">
        <span v-if="updatedAt">数据更新时间：{{ updatedAt }}</span>
        <span v-else>数据更新时间：—</span>
      </footer>
    </template>
  </section>
</template>

<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'

import { fetchJson } from '@/api/client'

type Overview = {
  cards: { label: string; value: number }[]
  modules: { name: string; created: number; pending: number; abnormal: number }[]
}

type CachedOverview = {
  fetchedAt: number
  overview: Overview
}

/**
 * 卡片与各模块列表共用后端 /api/overview 同一套汇总口径，这里只负责展示，
 * 绝不在取数失败时用本地样例顶替。
 */
const ENDPOINT = '/api/overview'
const CACHE_KEY = 'overview:last-success'

// 模块标识与侧边栏名称保持一致；只做文案映射，不参与任何统计。
const moduleLabels: Record<string, string> = {
  boiler: '锅炉设备',
  vessel: '压力容器',
  pressurepipe: '压力管道',
  crane: '起重机械',
  elevator: '电梯设备',
  forklift: '场内机动车辆',
  plan: '点检计划',
  spotcheck: '点检记录',
  lubricate: '润滑保养',
  inspect: '定期检验',
  report: '检验报告',
  hazard: '隐患登记',
  rectify: '整改闭环',
  register: '使用登记',
  operator: '作业人员',
  spare: '备件器材',
  contract: '维保合同',
  settle: '费用结算',
}

const cards = ref<Overview['cards']>([])
const moduleRows = ref<Overview['modules']>([])
const fetchedAt = ref<number | null>(null)
const usingCache = ref(false)
const loading = ref(false)
const errorMessage = ref('')

const hasData = computed(() => cards.value.length > 0 || moduleRows.value.length > 0)
const updatedAt = computed(() => (fetchedAt.value === null ? '' : formatTime(fetchedAt.value)))
const cacheLabel = computed(() => `上次成功取数的数据（${updatedAt.value}）`)
const showCacheNotice = computed(() => loading.value && usingCache.value)

function formatTime(ms: number): string {
  const date = new Date(ms)
  const pad = (value: number) => String(value).padStart(2, '0')
  return `${date.getFullYear()}-${pad(date.getMonth() + 1)}-${pad(date.getDate())} ${pad(date.getHours())}:${pad(date.getMinutes())}`
}

function isValidOverview(payload: unknown): payload is Overview {
  if (typeof payload !== 'object' || payload === null) {
    return false
  }
  const data = payload as Record<string, unknown>
  return Array.isArray(data.cards) && Array.isArray(data.modules)
}

/** 读取上次成功取数的缓存；重新打开页面时先渲染它，数字不再先跳样例再跳真实值。 */
function loadCache(): CachedOverview | null {
  try {
    const raw = window.localStorage.getItem(CACHE_KEY)
    if (!raw) {
      return null
    }
    const parsed = JSON.parse(raw) as CachedOverview
    if (
      typeof parsed.fetchedAt !== 'number' ||
      !isValidOverview(parsed.overview)
    ) {
      return null
    }
    return parsed
  } catch {
    return null
  }
}

function applyOverview(overview: Overview, time: number, fromCache: boolean) {
  cards.value = overview.cards
  moduleRows.value = overview.modules
  fetchedAt.value = time
  usingCache.value = fromCache
}

async function loadOverview() {
  loading.value = true
  errorMessage.value = ''
  try {
    const payload = await fetchJson<Overview>(ENDPOINT)
    if (!isValidOverview(payload)) {
      throw new Error('返回数据结构不完整')
    }
    const time = Date.now()
    applyOverview(payload, time, false)
    try {
      const cache: CachedOverview = { fetchedAt: time, overview: payload }
      window.localStorage.setItem(CACHE_KEY, JSON.stringify(cache))
    } catch {
      // 缓存写不进去（隐私模式/配额不足）不影响本次真实数据展示
    }
  } catch (error) {
    // 失败时保留已展示的数据（本次会话或缓存），绝不覆盖成写死的样例
    errorMessage.value = error instanceof Error ? error.message : '运营概览读取失败'
  } finally {
    loading.value = false
  }
}

function retry() {
  void loadOverview()
}

onMounted(() => {
  const cached = loadCache()
  if (cached) {
    applyOverview(cached.overview, cached.fetchedAt, true)
  }
  void loadOverview()
})
</script>

<style scoped>
.page-actions {
  display: flex;
  gap: 8px;
}

.state-panel {
  background: #fff;
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 32px 16px;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  margin-bottom: 12px;
  color: var(--muted);
  font-size: 13px;
}

.error-panel {
  border-color: #f0a8a0;
}

.alert-bar {
  background: #fff7ed;
  border: 1px solid #fed7aa;
  border-radius: 6px;
  padding: 8px 10px;
  font-size: 12px;
  margin-bottom: 12px;
}

.empty-state-text {
  margin: 0;
}
</style>
