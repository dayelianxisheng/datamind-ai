<script setup lang="ts">
import * as echarts from 'echarts'
import { nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'

type Dataset = {
  id: string
  name: string
  sheet_name: string | null
  row_count: number
  columns: { name: string; type: string; null_count: number }[]
  preview: { columns: string[]; rows: unknown[][] }
}

type Analysis = {
  answer: string
  question: string
  sql: string
  result: { columns: string[]; rows: unknown[][] }
  chart: null | { type: 'bar' | 'line'; title: string; x_field: string; y_field: string; unit: string }
  mode: string
}

const dataset = ref<Dataset | null>(null)
const analysis = ref<Analysis | null>(null)
const question = ref('哪个地区销售额最高？按销售额排序展示各地区')
const busy = ref(false)
const error = ref('')
const modelReady = ref(false)
const chartElement = ref<HTMLDivElement | null>(null)
let chart: echarts.ECharts | null = null

async function request<T>(url: string, options?: RequestInit): Promise<T> {
  const response = await fetch(url, options)
  const body = await response.json().catch(() => ({}))
  if (!response.ok) throw new Error(body.detail?.message || body.detail || '请求失败，请稍后重试。')
  return body
}

async function loadSample() {
  busy.value = true
  error.value = ''
  try {
    dataset.value = await request('/api/datasets/sample', { method: 'POST' })
    analysis.value = null
  } catch (reason) {
    error.value = reason instanceof Error ? reason.message : '示例数据加载失败。'
  } finally {
    busy.value = false
  }
}

async function upload(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (!file) return
  busy.value = true
  error.value = ''
  const form = new FormData()
  form.append('file', file)
  try {
    dataset.value = await request('/api/datasets', { method: 'POST', body: form })
    analysis.value = null
  } catch (reason) {
    error.value = reason instanceof Error ? reason.message : '文件上传失败。'
  } finally {
    busy.value = false
    input.value = ''
  }
}

async function analyze() {
  if (!dataset.value || !question.value.trim() || busy.value) return
  busy.value = true
  error.value = ''
  analysis.value = null
  try {
    analysis.value = await request('/api/analyses', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ dataset_id: dataset.value.id, question: question.value.trim() }),
    })
  } catch (reason) {
    error.value = reason instanceof Error ? reason.message : '分析失败。'
  } finally {
    busy.value = false
  }
}

function renderChart() {
  if (!chartElement.value || !analysis.value?.chart) return
  chart?.dispose()
  chart = echarts.init(chartElement.value)
  const spec = analysis.value.chart
  const rows = analysis.value.result.rows
  chart.setOption({
    color: ['#3569e8'],
    title: { text: spec.title, textStyle: { fontSize: 16, color: '#172033' } },
    tooltip: { trigger: 'axis', valueFormatter: (value: unknown) => `${Number(value).toLocaleString()} ${spec.unit}` },
    grid: { left: 50, right: 24, top: 58, bottom: 44, containLabel: true },
    xAxis: { type: 'category', data: rows.map((row) => row[0]), axisLabel: { color: '#65718a' } },
    yAxis: { type: 'value', name: spec.unit, axisLabel: { color: '#65718a' }, splitLine: { lineStyle: { color: '#edf0f5' } } },
    series: [
      {
        type: spec.type,
        data: rows.map((row) => row[1]),
        smooth: spec.type === 'line',
        barMaxWidth: 48,
        areaStyle: spec.type === 'line' ? { opacity: 0.08 } : undefined,
      },
    ],
  })
}

function formatValue(value: unknown) {
  return typeof value === 'number' ? value.toLocaleString() : String(value ?? '—')
}

watch(analysis, async () => {
  await nextTick()
  renderChart()
})

onMounted(async () => {
  try {
    const health = await request<{ model_configured: boolean }>('/api/health')
    modelReady.value = health.model_configured
  } catch {
    modelReady.value = false
  }
  await loadSample()
  window.addEventListener('resize', () => chart?.resize())
})

onBeforeUnmount(() => chart?.dispose())
</script>

<template>
  <main>
    <aside>
      <div class="brand"><span>◆</span> DataMind AI</div>
      <p class="muted">智能数据分析工作台</p>

      <label class="upload" :class="{ disabled: busy }">
        <input type="file" accept=".csv,.xlsx" :disabled="busy" @change="upload" />
        <span>上传 CSV / Excel</span>
        <small>最大 10 MB</small>
      </label>
      <button class="sample" :disabled="busy" @click="loadSample">使用示例数据</button>

      <template v-if="dataset">
        <div class="dataset-title">
          <span class="file-icon">▦</span>
          <div><strong>{{ dataset.name }}</strong><small>{{ dataset.row_count.toLocaleString() }} 行 · {{ dataset.columns.length }} 列</small></div>
        </div>
        <div class="fields">
          <p>字段结构</p>
          <div v-for="column in dataset.columns" :key="column.name">
            <span>{{ column.name }}</span><small>{{ column.type }}</small>
          </div>
        </div>
      </template>
    </aside>

    <section>
      <header>
        <div>
          <p class="eyebrow">CONVERSATIONAL ANALYTICS</p>
          <h1>用一句话，读懂你的数据</h1>
          <p class="subtitle">上传表格，用自然语言完成查询、洞察和可视化。</p>
        </div>
        <span class="status" :class="{ offline: !modelReady }"><i></i>{{ modelReady ? 'DeepSeek 已连接' : '模型未连接' }}</span>
      </header>

      <div v-if="dataset" class="preview card">
        <div class="card-head"><strong>数据预览</strong><span>{{ dataset.sheet_name ? `工作表：${dataset.sheet_name}` : dataset.name }}</span></div>
        <div class="table-wrap">
          <table>
            <thead><tr><th v-for="column in dataset.preview.columns" :key="column">{{ column }}</th></tr></thead>
            <tbody><tr v-for="(row, index) in dataset.preview.rows.slice(0, 5)" :key="index"><td v-for="(value, cell) in row" :key="cell">{{ formatValue(value) }}</td></tr></tbody>
          </table>
        </div>
      </div>

      <div class="suggestions">
        <button @click="question = '哪个地区销售额最高？按销售额排序展示各地区'">地区销售排名</button>
        <button @click="question = '2026 年每个月的总销售额趋势如何？'">月度销售趋势</button>
        <button @click="question = '各地区的平均客单价是多少？按从高到低排序'">平均客单价</button>
      </div>

      <div class="ask card">
        <textarea v-model="question" rows="2" placeholder="例如：哪个地区销售额最高？" @keydown.ctrl.enter="analyze"></textarea>
        <button :disabled="!dataset || !question.trim() || busy" @click="analyze">
          {{ busy ? '正在分析…' : '开始分析' }}
        </button>
      </div>

      <p v-if="error" class="error">{{ error }}</p>

      <article v-if="analysis" class="result card">
        <div class="result-label"><span>AI 分析结论</span><small>数据由 MySQL 实际计算</small></div>
        <h2>{{ analysis.answer }}</h2>
        <div v-if="analysis.chart" ref="chartElement" class="chart"></div>
        <div class="table-wrap result-table">
          <table>
            <thead><tr><th v-for="column in analysis.result.columns" :key="column">{{ column }}</th></tr></thead>
            <tbody><tr v-for="(row, index) in analysis.result.rows" :key="index"><td v-for="(value, cell) in row" :key="cell">{{ formatValue(value) }}</td></tr></tbody>
          </table>
        </div>
        <details v-if="analysis.sql"><summary>查看 SQL 计算依据</summary><pre>{{ analysis.sql }}</pre></details>
      </article>
    </section>
  </main>
</template>

<style>
:root { color-scheme: light; font-family: Inter, ui-sans-serif, system-ui, -apple-system, "Segoe UI", sans-serif; }
* { box-sizing: border-box; }
body { margin: 0; background: #f5f7fb; color: #172033; }
button, textarea, input { font: inherit; }
button { cursor: pointer; }
button:disabled { cursor: not-allowed; opacity: .55; }
main { display: grid; grid-template-columns: 270px minmax(0, 1fr); min-height: 100vh; }
aside { position: sticky; top: 0; height: 100vh; padding: 28px 22px; overflow-y: auto; color: #fff; background: linear-gradient(180deg, #10244c 0%, #101d38 100%); }
.brand { display: flex; gap: 10px; align-items: center; font-size: 20px; font-weight: 760; }
.brand span { color: #58d0c3; }
.muted { margin: 5px 0 28px 30px; color: #91a4c5; font-size: 12px; }
.upload { display: grid; place-items: center; gap: 3px; padding: 19px 12px; border: 1px dashed #5b7199; border-radius: 12px; color: #dfe8fa; cursor: pointer; background: #ffffff09; }
.upload:hover { background: #ffffff10; }
.upload input { display: none; }
.upload small { color: #8497b9; }
.sample { width: 100%; margin-top: 10px; padding: 10px; color: #cbd7ed; background: transparent; border: 1px solid #344b72; border-radius: 10px; }
.dataset-title { display: flex; gap: 11px; align-items: center; margin-top: 30px; padding: 14px 0; border-bottom: 1px solid #2a3d60; }
.dataset-title div { min-width: 0; }
.dataset-title strong, .dataset-title small { display: block; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.dataset-title small { margin-top: 3px; color: #91a4c5; }
.file-icon { display: grid; place-items: center; width: 34px; height: 34px; border-radius: 8px; background: #3569e8; }
.fields > p { margin-top: 22px; color: #8497b9; font-size: 12px; text-transform: uppercase; letter-spacing: .08em; }
.fields > div { display: flex; justify-content: space-between; gap: 10px; padding: 8px 0; color: #dbe5f7; }
.fields small { color: #7f93b4; }
section { width: min(1100px, calc(100% - 64px)); margin: 0 auto; padding: 52px 0 80px; }
header { display: flex; justify-content: space-between; gap: 24px; align-items: flex-start; }
.eyebrow { margin: 0 0 9px; color: #3569e8; font-size: 11px; font-weight: 800; letter-spacing: .14em; }
h1 { margin: 0; font-size: clamp(34px, 5vw, 52px); letter-spacing: -.045em; }
.subtitle { margin: 10px 0 28px; color: #6c768a; font-size: 16px; }
.status { display: flex; gap: 7px; align-items: center; flex: none; padding: 8px 12px; border: 1px solid #cae9df; border-radius: 99px; color: #20765f; background: #effaf6; font-size: 12px; }
.status i { width: 7px; height: 7px; border-radius: 50%; background: #28a47f; }
.status.offline { color: #9b4f1e; border-color: #f0d8c4; background: #fff8f1; }
.status.offline i { background: #dc7d34; }
.card { border: 1px solid #e2e7f0; border-radius: 16px; background: #fff; box-shadow: 0 10px 36px #19305a0b; }
.preview { margin-top: 8px; overflow: hidden; }
.card-head { display: flex; justify-content: space-between; padding: 17px 20px; border-bottom: 1px solid #edf0f5; }
.card-head span { color: #7a8498; font-size: 12px; }
.table-wrap { overflow-x: auto; }
table { width: 100%; border-collapse: collapse; font-size: 13px; }
th, td { padding: 10px 18px; text-align: left; white-space: nowrap; border-bottom: 1px solid #edf0f5; }
th { color: #637087; background: #f8f9fc; font-weight: 650; }
td { color: #344057; }
.suggestions { display: flex; flex-wrap: wrap; gap: 8px; margin: 20px 0 10px; }
.suggestions button { padding: 8px 12px; color: #526079; border: 1px solid #dce2ec; border-radius: 99px; background: #fff; }
.suggestions button:hover { color: #2858bf; border-color: #adc2ee; }
.ask { display: flex; align-items: flex-end; gap: 12px; padding: 14px; }
.ask textarea { flex: 1; min-height: 50px; resize: vertical; padding: 9px 10px; color: #172033; border: 0; outline: none; }
.ask > button { padding: 12px 22px; color: #fff; border: 0; border-radius: 10px; background: #285fd3; font-weight: 700; }
.error { padding: 12px 15px; color: #a33229; border: 1px solid #f0cbc7; border-radius: 10px; background: #fff4f2; }
.result { margin-top: 20px; padding: 24px; }
.result-label { display: flex; justify-content: space-between; color: #3569e8; font-weight: 750; }
.result-label small { color: #8290a7; font-weight: 500; }
.result h2 { margin: 9px 0 18px; font-size: 22px; line-height: 1.45; }
.chart { width: 100%; height: 360px; margin-top: 12px; }
.result-table { margin-top: 15px; border: 1px solid #e6eaf1; border-radius: 10px; }
.result-table tbody tr:last-child td { border-bottom: 0; }
details { margin-top: 18px; color: #5f6c82; }
summary { cursor: pointer; }
pre { overflow-x: auto; padding: 14px; color: #dce6f7; border-radius: 10px; background: #17233c; white-space: pre-wrap; }
@media (max-width: 760px) {
  main { grid-template-columns: 1fr; }
  aside { position: static; height: auto; }
  .fields { display: none; }
  section { width: min(100% - 30px, 1100px); padding-top: 32px; }
  header { display: block; }
  .status { width: fit-content; margin-bottom: 20px; }
  .ask { display: block; }
  .ask textarea, .ask > button { width: 100%; }
  .ask > button { margin-top: 8px; }
}
</style>
