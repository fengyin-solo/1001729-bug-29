<template>
  <section class="page">
    <header class="page-head">
      <div>
        <h2>运营概览</h2>
        <p class="page-desc">汇总各业务模块的关键指标，先看总量再看异常。</p>
      </div>
      <div class="page-actions">
        <button class="btn" type="button" :disabled="loading" @click="load">
          {{ loading ? '正在加载…' : '刷新' }}
        </button>
      </div>
    </header>

    <p v-if="loading" class="empty-state">运营概览加载中…</p>

    <div v-else-if="errorMessage" class="overview-error">
      <span class="error-text">运营概览读取失败：{{ errorMessage }}。为避免误读，未展示任何统计数字。</span>
      <button class="btn" type="button" @click="load">重试</button>
    </div>

    <template v-else>
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
            <td>{{ row.name }}</td>
            <td>{{ row.created }}</td>
            <td>{{ row.pending }}</td>
            <td>{{ row.abnormal }}</td>
          </tr>
          <tr v-if="!moduleRows.length">
            <td colspan="4" class="empty-state">暂无运营数据，可点击右上角「刷新」重新获取</td>
          </tr>
        </tbody>
      </table>
    </template>
  </section>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue'

import { fetchJson } from '@/api/client'

type Overview = {
  cards: { label: string; value: number }[]
  modules: { name: string; created: number; pending: number; abnormal: number }[]
}

const cards = ref<Overview['cards']>([])
const moduleRows = ref<Overview['modules']>([])
const loading = ref(false)
const errorMessage = ref('')

async function load() {
  loading.value = true
  errorMessage.value = ''
  try {
    const payload = await fetchJson<Overview>('/api/overview')
    cards.value = payload.cards ?? []
    moduleRows.value = payload.modules ?? []
  } catch (error) {
    // 取数失败时清空卡片与列表，只展示错误说明，绝不回落到内置样例数字
    cards.value = []
    moduleRows.value = []
    errorMessage.value = error instanceof Error ? error.message : '接口请求失败'
  } finally {
    loading.value = false
  }
}

onMounted(load)
</script>

<style scoped>
.overview-error {
  display: flex;
  align-items: center;
  gap: 12px;
  background: #fff;
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 16px;
}
</style>
