<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Review demand forecasts and place restocking orders within your budget.</p>
    </div>

    <div v-if="loading" class="loading">Loading forecasts...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Slider -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-controls">
          <input
            type="range"
            min="0"
            max="50000"
            step="500"
            v-model.number="budget"
            class="budget-slider"
          />
          <span class="budget-value">${{ budget.toLocaleString() }}</span>
        </div>
      </div>

      <!-- Success Banner -->
      <div v-if="successOrder" class="success-banner">
        <div class="success-content">
          <strong>Order {{ successOrder.order_number }} placed successfully.</strong>
          Expected delivery: {{ formatDate(successOrder.expected_delivery) }}.
        </div>
        <button class="place-another-btn" @click="resetOrder">Place Another Order</button>
      </div>

      <!-- Recommended Items Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
        </div>
        <div v-if="recommendedItems.length === 0" class="empty-state">
          No items fit within the current budget. Increase the budget slider to see recommendations.
        </div>
        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th class="col-num">Forecasted Demand</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-num">Order Qty</th>
                <th class="col-num">Total Cost</th>
                <th>Trend</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.id">
                <td><code class="sku">{{ item.item_sku }}</code></td>
                <td>{{ item.item_name }}</td>
                <td class="col-num">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-num">${{ item.unit_cost.toFixed(2) }}</td>
                <td class="col-num">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-num"><strong>${{ (item.unit_cost * item.forecasted_demand).toLocaleString() }}</strong></td>
                <td>
                  <span :class="['badge', item.trend.toLowerCase()]">{{ item.trend }}</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Summary Row -->
        <div class="order-summary" v-if="recommendedItems.length > 0">
          <span class="summary-text">
            {{ recommendedItems.length }} item{{ recommendedItems.length !== 1 ? 's' : '' }} selected
            &nbsp;&middot;&nbsp;
            Total: <strong>${{ totalCost.toLocaleString() }}</strong>
            &nbsp;&middot;&nbsp;
            Remaining budget: <strong>${{ remainingBudget.toLocaleString() }}</strong>
          </span>
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0 || submitting || !!successOrder"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>

        <div v-if="submitError" class="error submit-error">{{ submitError }}</div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentLocale } = useI18n()

    const forecasts = ref([])
    const loading = ref(true)
    const error = ref(null)
    const budget = ref(10000)
    const submitting = ref(false)
    const submitError = ref(null)
    const successOrder = ref(null)

    const loadForecasts = async () => {
      loading.value = true
      error.value = null
      try {
        forecasts.value = await api.getRestockingForecasts()
      } catch (err) {
        error.value = 'Failed to load restocking forecasts: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Greedy selection: walk sorted-by-forecasted_demand list, add items until budget exceeded
    const recommendedItems = computed(() => {
      let remaining = budget.value
      const selected = []
      for (const item of forecasts.value) {
        const cost = item.unit_cost * item.forecasted_demand
        if (cost === 0) continue
        if (cost <= remaining) {
          selected.push(item)
          remaining -= cost
        }
      }
      return selected
    })

    const totalCost = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.unit_cost * item.forecasted_demand, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    const formatDate = (dateString) => {
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString(locale, {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || submitting.value) return
      submitting.value = true
      submitError.value = null
      try {
        const items = recommendedItems.value.map(item => ({
          item_sku: item.item_sku,
          item_name: item.item_name,
          quantity: item.forecasted_demand,
          unit_cost: item.unit_cost
        }))
        const result = await api.createRestockingOrder({ items, budget: budget.value })
        successOrder.value = result
      } catch (err) {
        submitError.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    const resetOrder = () => {
      successOrder.value = null
      submitError.value = null
    }

    onMounted(loadForecasts)

    return {
      forecasts,
      loading,
      error,
      budget,
      submitting,
      submitError,
      successOrder,
      recommendedItems,
      totalCost,
      remainingBudget,
      formatDate,
      placeOrder,
      resetOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

.budget-card {
  margin-bottom: 1.25rem;
}

.budget-controls {
  display: flex;
  align-items: center;
  gap: 1.5rem;
}

.budget-slider {
  flex: 1;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: box-shadow 0.2s;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 1px 8px rgba(37, 99, 235, 0.6);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-slider::-webkit-slider-runnable-track {
  background: linear-gradient(to right, #2563eb var(--progress, 20%), #e2e8f0 var(--progress, 20%));
  border-radius: 3px;
  height: 6px;
}

.budget-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 120px;
  text-align: right;
}

.success-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 10px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.25rem;
  color: #065f46;
}

.success-content {
  font-size: 0.938rem;
}

.place-another-btn {
  background: white;
  color: #065f46;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 0.5rem 1.25rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
  white-space: nowrap;
}

.place-another-btn:hover {
  background: #ecfdf5;
}

.restocking-table {
  width: 100%;
  border-collapse: collapse;
}

.col-num {
  text-align: right;
}

.sku {
  font-family: 'Menlo', 'Monaco', 'Consolas', monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
}

.empty-state {
  padding: 2.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.order-summary {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.875rem 0.75rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 0.5rem;
  background: #f8fafc;
  border-radius: 0 0 8px 8px;
}

.summary-text {
  font-size: 0.875rem;
  color: #475569;
}

.summary-text strong {
  color: #0f172a;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border-radius: 8px;
  padding: 0.625rem 1.5rem;
  font-weight: 600;
  border: none;
  cursor: pointer;
  font-size: 0.875rem;
  transition: background 0.15s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.submit-error {
  margin: 0.75rem 0.75rem 0;
}
</style>
