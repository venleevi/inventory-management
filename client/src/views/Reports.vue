<template>
  <div class="reports">
    <div class="page-header">
      <h2>{{ t('reports.title') }}</h2>
      <p>{{ t('reports.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('reports.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else-if="quarterlyData.length === 0 && monthlyData.length === 0" class="loading">
      {{ t('reports.noData') }}
    </div>
    <div v-else>
      <!-- Quarterly Performance -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('reports.quarterlyPerformance') }}</h3>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('reports.quarter') }}</th>
                <th>{{ t('reports.totalOrders') }}</th>
                <th>{{ t('reports.totalRevenue') }}</th>
                <th>{{ t('reports.avgOrderValue') }}</th>
                <th>{{ t('reports.fulfillmentRate') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="q in quarterlyData" :key="q.quarter">
                <td><strong>{{ q.quarter }}</strong></td>
                <td>{{ q.total_orders }}</td>
                <td>{{ money(q.total_revenue) }}</td>
                <td>{{ money(q.avg_order_value) }}</td>
                <td>
                  <span :class="['badge', getFulfillmentClass(q.fulfillment_rate)]">
                    {{ q.fulfillment_rate }}%
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Monthly Trends Chart -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('reports.monthlyRevenueTrend') }}</h3>
        </div>
        <div class="chart-container">
          <div class="bar-chart">
            <div v-for="row in monthRows" :key="row.month" class="bar-wrapper">
              <div class="bar-container">
                <div
                  class="bar"
                  :style="{ height: row.barHeight + 'px' }"
                  :title="money(row.revenue)"
                ></div>
              </div>
              <div class="bar-label">{{ row.label }}</div>
            </div>
          </div>
        </div>
      </div>

      <!-- Month-over-Month Comparison -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('reports.monthOverMonth') }}</h3>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('reports.month') }}</th>
                <th>{{ t('reports.orders') }}</th>
                <th>{{ t('reports.revenue') }}</th>
                <th>{{ t('reports.change') }}</th>
                <th>{{ t('reports.growthRate') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="row in monthRows" :key="row.month">
                <td><strong>{{ row.label }}</strong></td>
                <td>{{ row.order_count }}</td>
                <td>{{ money(row.revenue) }}</td>
                <td>
                  <span v-if="row.hasPrevious" :class="row.changeClass">{{ row.changeText }}</span>
                  <span v-else>-</span>
                </td>
                <td>
                  <span v-if="row.hasPrevious" :class="row.changeClass">{{ row.growthText }}</span>
                  <span v-else>-</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Summary Stats -->
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.totalRevenueYtd') }}</div>
          <div class="stat-value">{{ money(totalRevenue) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.avgMonthlyRevenue') }}</div>
          <div class="stat-value">{{ money(avgMonthlyRevenue) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.totalOrdersYtd') }}</div>
          <div class="stat-value">{{ totalOrders }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.bestQuarter') }}</div>
          <div class="stat-value">{{ bestQuarter }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'
import { formatCurrencyWithDecimals } from '../utils/currency'

const { t, currentCurrency, currentLocale } = useI18n()
const { selectedPeriod, selectedLocation, selectedCategory, selectedStatus, getCurrentFilters } = useFilters()

const loading = ref(true)
const error = ref(null)
const quarterlyData = ref([])
const monthlyData = ref([])

const MONTH_KEYS = ['jan', 'feb', 'mar', 'apr', 'may', 'jun', 'jul', 'aug', 'sep', 'oct', 'nov', 'dec']

const money = (amount) => formatCurrencyWithDecimals(amount ?? 0, currentCurrency.value, 2)

// Convert YYYY-MM into a localized label; falls back to the raw string if invalid
const formatMonth = (monthStr) => {
  const date = new Date(`${monthStr}-01T00:00:00`)
  if (isNaN(date.getTime())) return monthStr
  const name = t(`months.${MONTH_KEYS[date.getMonth()]}`)
  const year = date.getFullYear()
  return currentLocale.value === 'ja' ? `${year}年${name}` : `${name} ${year}`
}

const getFulfillmentClass = (rate) => {
  if (rate >= 90) return 'success'
  if (rate >= 75) return 'warning'
  return 'danger'
}

const MAX_BAR_HEIGHT = 200

const maxRevenue = computed(() => Math.max(0, ...monthlyData.value.map(m => m.revenue)))

// Per-month rows with derived change/growth values
const monthRows = computed(() => {
  return monthlyData.value.map((m, i) => {
    const prev = i > 0 ? monthlyData.value[i - 1] : null
    const diff = prev ? m.revenue - prev.revenue : 0
    let changeText = money(0)
    if (diff > 0) changeText = `+${money(diff)}`
    else if (diff < 0) changeText = `-${money(Math.abs(diff))}`

    let growthText = t('reports.notAvailable')
    if (prev && prev.revenue !== 0) {
      const rate = (diff / prev.revenue) * 100
      growthText = `${rate > 0 ? '+' : ''}${rate.toFixed(1)}%`
    }

    return {
      ...m,
      label: formatMonth(m.month),
      barHeight: maxRevenue.value === 0 ? 0 : (m.revenue / maxRevenue.value) * MAX_BAR_HEIGHT,
      hasPrevious: prev !== null,
      changeText,
      growthText,
      changeClass: diff > 0 ? 'positive-change' : diff < 0 ? 'negative-change' : ''
    }
  })
})

const totalRevenue = computed(() => monthlyData.value.reduce((sum, m) => sum + m.revenue, 0))

const avgMonthlyRevenue = computed(() => {
  return monthlyData.value.length > 0 ? totalRevenue.value / monthlyData.value.length : 0
})

const totalOrders = computed(() => monthlyData.value.reduce((sum, m) => sum + m.order_count, 0))

const bestQuarter = computed(() => {
  let best = null
  for (const q of quarterlyData.value) {
    if (!best || q.total_revenue > best.total_revenue) best = q
  }
  return best && best.total_revenue > 0 ? best.quarter : '-'
})

// Ignore responses from superseded requests when filters change quickly
let requestId = 0

const loadData = async () => {
  const currentRequest = ++requestId
  loading.value = true
  error.value = null
  try {
    const filters = getCurrentFilters()
    const [quarterly, monthly] = await Promise.all([
      api.getQuarterlyReports(filters),
      api.getMonthlyTrends(filters)
    ])
    if (currentRequest !== requestId) return
    quarterlyData.value = quarterly
    monthlyData.value = monthly
  } catch (err) {
    if (currentRequest !== requestId) return
    error.value = `${t('reports.loadError')}: ${err.message}`
    console.error('Failed to load reports:', err)
  } finally {
    if (currentRequest === requestId) loading.value = false
  }
}

watch([selectedPeriod, selectedLocation, selectedCategory, selectedStatus], loadData)

onMounted(loadData)
</script>

<style scoped>
.chart-container {
  padding: var(--space-8) var(--space-4);
  min-height: 300px;
}

.bar-chart {
  display: flex;
  align-items: flex-end;
  justify-content: space-around;
  height: 250px;
  gap: var(--space-2);
}

.bar-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  flex: 1;
  max-width: 80px;
}

.bar-container {
  height: 200px;
  display: flex;
  align-items: flex-end;
  width: 100%;
}

.bar {
  width: 100%;
  background: var(--color-accent);
  opacity: 0.85;
  border-radius: var(--space-1) var(--space-1) 0 0;
  transition: opacity 0.15s ease;
  cursor: pointer;
}

.bar:hover {
  opacity: 1;
}

.bar-label {
  font-size: 12px;
  color: var(--color-muted);
  text-align: center;
  transform: rotate(-45deg);
  white-space: nowrap;
  margin-top: var(--space-6);
}

.positive-change {
  color: var(--color-success);
  font-weight: 600;
}

.negative-change {
  color: var(--color-danger);
  font-weight: 600;
}
</style>
