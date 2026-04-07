<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <!-- Budget Card -->
    <div class="card">
      <div class="card-header"><h3 class="card-title">{{ t('restocking.budgetConfig') }}</h3></div>
      <div class="budget-section">
        <div class="budget-display">
          <span class="budget-label">{{ t('restocking.availableBudget') }}</span>
          <span class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>
        <input type="range" min="0" max="50000" step="500" v-model.number="budget" class="budget-slider" />
        <div class="budget-range-labels">
          <span>{{ currencySymbol }}0</span>
          <span>{{ currencySymbol }}50,000</span>
        </div>
      </div>
    </div>

    <!-- Stats Row -->
    <div class="stats-grid">
      <div class="stat-card">
        <div class="stat-label">{{ t('restocking.totalCost') }}</div>
        <div class="stat-value">{{ currencySymbol }}{{ Math.round(totalSelectedCost).toLocaleString() }}</div>
      </div>
      <div class="stat-card">
        <div class="stat-label">{{ t('restocking.remainingBudget') }}</div>
        <div class="stat-value" :class="remainingBudget < 0 ? 'text-danger' : ''">{{ currencySymbol }}{{ Math.round(remainingBudget).toLocaleString() }}</div>
      </div>
      <div class="stat-card">
        <div class="stat-label">{{ t('restocking.itemsToRestock') }}</div>
        <div class="stat-value">{{ selectedItems.length }}</div>
      </div>
    </div>

    <!-- Success Banner -->
    <div v-if="orderSubmitted" class="success-banner">
      <strong>{{ t('restocking.orderPlaced') }}</strong>
      Order {{ lastOrderNumber }} — Expected delivery: {{ lastExpectedDelivery }}
      <button class="banner-close" @click="orderSubmitted = false">×</button>
    </div>

    <!-- Recommendations Card -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
        <button
          class="btn-primary"
          :disabled="selectedItems.length === 0 || isSubmitting"
          @click="placeOrder"
        >
          {{ isSubmitting ? t('common.loading') : t('restocking.placeOrder') }}
        </button>
      </div>

      <div v-if="loading">{{ t('common.loading') }}</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else-if="recommendations.length === 0" class="no-data">{{ t('restocking.noItems') }}</div>
      <div v-else class="table-container">
        <table class="recommendations-table">
          <thead>
            <tr>
              <th>{{ t('restocking.table.sku') }}</th>
              <th>{{ t('restocking.table.itemName') }}</th>
              <th>{{ t('restocking.table.trend') }}</th>
              <th class="text-right">{{ t('restocking.table.forecastedDemand') }}</th>
              <th class="text-right">{{ t('restocking.table.onHand') }}</th>
              <th class="text-right">{{ t('restocking.table.gap') }}</th>
              <th class="text-right">{{ t('restocking.table.unitCost') }}</th>
              <th class="text-right">{{ t('restocking.table.lineCost') }}</th>
              <th class="text-center">{{ t('restocking.table.include') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="item in recommendations"
              :key="item.sku"
              :class="{ 'row-selected': item.selected && item.gap > 0, 'row-muted': item.gap === 0 }"
            >
              <td class="sku-cell">{{ item.sku }}</td>
              <td>{{ item.name }}</td>
              <td><span :class="['badge', getTrendClass(item.trend)]">{{ item.trend }}</span></td>
              <td class="text-right">{{ item.forecasted_demand.toLocaleString() }}</td>
              <td class="text-right">{{ item.quantity_on_hand.toLocaleString() }}</td>
              <td class="text-right">{{ item.gap.toLocaleString() }}</td>
              <td class="text-right">{{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</td>
              <td class="text-right"><strong>{{ currencySymbol }}{{ item.line_cost.toLocaleString() }}</strong></td>
              <td class="text-center">
                <input
                  type="checkbox"
                  :checked="item.selected"
                  :disabled="item.gap === 0"
                  @change="item.selected = $event.target.checked"
                />
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { useI18n } from '../composables/useI18n'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const budget = ref(10000)
    const loading = ref(false)
    const error = ref(null)
    const recommendations = ref([])
    const isSubmitting = ref(false)
    const orderSubmitted = ref(false)
    const lastOrderNumber = ref('')
    const lastExpectedDelivery = ref('')

    const selectedItems = computed(() =>
      recommendations.value.filter(r => r.selected && r.gap > 0)
    )

    const totalSelectedCost = computed(() =>
      selectedItems.value.reduce((sum, item) => sum + item.line_cost, 0)
    )

    const remainingBudget = computed(() => budget.value - totalSelectedCost.value)

    const currencySymbol = computed(() =>
      currentCurrency.value === 'JPY' ? '¥' : '$'
    )

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRecommendations(budget.value)
        recommendations.value = data.candidates
      } catch (err) {
        error.value = 'Failed to load recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      isSubmitting.value = true
      try {
        const result = await api.createRestockOrder({
          items: selectedItems.value.map(item => ({
            sku: item.sku,
            name: item.name,
            quantity: item.gap,
            unit_price: item.unit_cost
          })),
          total_value: Math.round(totalSelectedCost.value * 100) / 100,
          budget: budget.value
        })
        lastOrderNumber.value = result.order_number
        lastExpectedDelivery.value = result.expected_delivery
        orderSubmitted.value = true
        recommendations.value.forEach(r => { r.selected = false })
      } catch (err) {
        error.value = 'Failed to place order'
        console.error(err)
      } finally {
        isSubmitting.value = false
      }
    }

    const getTrendClass = (trend) => {
      if (trend === 'increasing') return 'success'
      if (trend === 'decreasing') return 'danger'
      return 'info'
    }

    let debounceTimer = null
    watch(budget, () => {
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        loadRecommendations()
      }, 400)
    })

    onMounted(() => loadRecommendations())

    return {
      t,
      budget,
      loading,
      error,
      recommendations,
      isSubmitting,
      orderSubmitted,
      lastOrderNumber,
      lastExpectedDelivery,
      selectedItems,
      totalSelectedCost,
      remainingBudget,
      currencySymbol,
      loadRecommendations,
      placeOrder,
      getTrendClass
    }
  }
}
</script>

<style scoped>
.budget-section {
  padding: 1.5rem;
}
.budget-display {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}
.budget-label {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}
.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}
.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
  margin-bottom: 0.5rem;
}
.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
}
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  color: #065f46;
  font-size: 0.875rem;
  display: flex;
  align-items: center;
  gap: 1rem;
  position: relative;
}
.banner-close {
  margin-left: auto;
  background: none;
  border: none;
  font-size: 1.25rem;
  color: #065f46;
  cursor: pointer;
  line-height: 1;
}
.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
}
.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
.btn-primary:not(:disabled):hover {
  background: #1d4ed8;
}
.recommendations-table {
  width: 100%;
  border-collapse: collapse;
}
.recommendations-table th,
.recommendations-table td {
  padding: 0.75rem 1rem;
  text-align: left;
  border-bottom: 1px solid #f1f5f9;
  font-size: 0.875rem;
}
.recommendations-table th {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  background: #f8fafc;
}
.row-selected {
  background: #eff6ff;
}
.row-muted td {
  color: #94a3b8;
}
.sku-cell {
  font-family: monospace;
  font-size: 0.813rem;
  color: #475569;
}
.text-right { text-align: right; }
.text-center { text-align: center; }
.no-data {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.875rem;
}
.text-danger { color: #ef4444; }
</style>
