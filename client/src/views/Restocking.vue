<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Set a budget to get AI-recommended restock orders based on demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Success banner (shown after a successful order placement) -->
      <div v-if="successOrder" class="success-banner">
        <span>
          Order <strong>{{ successOrder.order_number }}</strong> placed —
          expected delivery in {{ successOrder.lead_time_days }} days
          ({{ formatDate(successOrder.expected_delivery) }})
        </span>
        <button class="dismiss-btn" @click="successOrder = null">&times;</button>
      </div>

      <!-- Budget Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-body">
          <div class="slider-row">
            <span class="budget-label">Budget: {{ formatCurrency(budget) }}</span>
            <input
              type="range"
              class="budget-slider"
              :min="0"
              :max="maxBudget"
              :step="500"
              v-model.number="budget"
            />
          </div>
          <!-- Progress bar: proportion of budget already allocated to recommendations -->
          <div class="progress-track">
            <div
              class="progress-fill"
              :style="{ width: budgetUsedPercent + '%' }"
            ></div>
          </div>
          <p class="budget-caption">
            Allocating {{ formatCurrency(selectedTotal) }} of {{ formatCurrency(budget) }} budget
            ({{ budgetUsedPercent }}%)
          </p>
        </div>
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Restock Items ({{ recommendations.length }} items)</h3>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Category</th>
                <th>Trend</th>
                <th>Order Qty</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <!-- Empty state when no items fit the budget -->
              <tr v-if="recommendations.length === 0">
                <td colspan="7" class="empty-state">
                  No items fit within the current budget. Increase the budget to see recommendations.
                </td>
              </tr>
              <tr v-for="item in recommendations" :key="item.id">
                <td><code class="sku">{{ item.item_sku }}</code></td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.category }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.gap }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(item.line_total) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Table footer: summary + action button -->
        <div class="table-footer">
          <span class="footer-summary">
            {{ recommendations.length }} items selected &middot; {{ formatCurrency(selectedTotal) }} total
          </span>
          <button
            class="place-order-btn"
            :disabled="recommendations.length === 0 || placing"
            @click="placeOrder"
          >
            {{ placing ? 'Placing...' : 'Place Order' }}
          </button>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

// Trend sort priority: increasing items are most urgent, decreasing least
const TREND_ORDER = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const { t } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const demandItems = ref([])
    const budget = ref(0)
    const placing = ref(false)
    const successOrder = ref(null)
    // Guard to ensure we only auto-initialise the budget once after data loads
    const budgetInitialized = ref(false)

    // --- Derived / computed ---

    /**
     * Enriched items: only items where demand is forecast to grow.
     * Sorted by trend urgency (increasing → stable → decreasing).
     */
    const enrichedItems = computed(() => {
      return demandItems.value
        .filter(item => item.forecasted_demand > item.current_demand)
        .map(item => {
          const gap = item.forecasted_demand - item.current_demand
          return {
            ...item,
            gap,
            line_total: gap * item.unit_cost
          }
        })
        .sort((a, b) => (TREND_ORDER[a.trend] ?? 99) - (TREND_ORDER[b.trend] ?? 99))
    })

    /**
     * Maximum possible budget is the sum of all enriched items' line totals.
     * Default to 50000 while data is still loading so the slider has a sensible range.
     */
    const maxBudget = computed(() => {
      if (enrichedItems.value.length === 0) return 50000
      return enrichedItems.value.reduce((sum, item) => sum + item.line_total, 0)
    })

    /**
     * Greedy selection: walk sorted enriched items in order and include each
     * item only if its full line_total fits in the remaining budget.
     */
    const recommendations = computed(() => {
      let remaining = budget.value
      const result = []
      for (const item of enrichedItems.value) {
        if (item.line_total <= remaining) {
          result.push(item)
          remaining -= item.line_total
        }
      }
      return result
    })

    const selectedTotal = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.line_total, 0)
    )

    const budgetUsedPercent = computed(() => {
      if (budget.value === 0) return 0
      return Math.min(100, Math.round((selectedTotal.value / budget.value) * 100))
    })

    // --- Watchers ---

    /**
     * Once enrichedItems has data, set the default budget to half of the
     * maximum. Only runs on the first load so the user's manual adjustments
     * are not reset if data somehow re-evaluates.
     */
    watch(enrichedItems, (items) => {
      if (items.length > 0 && !budgetInitialized.value) {
        budget.value = Math.round(maxBudget.value / 2)
        budgetInitialized.value = true
      }
    })

    // --- Helpers ---

    const formatCurrency = (value) =>
      value.toLocaleString('en-US', { style: 'currency', currency: 'USD' })

    const formatDate = (dateString) => {
      const d = new Date(dateString)
      if (isNaN(d.getTime())) return dateString
      return d.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })
    }

    // --- Data loading ---

    const loadDemand = async () => {
      try {
        loading.value = true
        error.value = null
        demandItems.value = await api.getDemandForecasts()
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // --- Order placement ---

    const placeOrder = async () => {
      if (recommendations.value.length === 0 || placing.value) return
      placing.value = true
      try {
        const items = recommendations.value.map(item => ({
          sku: item.item_sku,
          name: item.item_name,
          quantity: item.gap,
          unit_price: item.unit_cost,
          category: item.category
        }))
        const order = await api.createRestockingOrder({
          items,
          total_cost: selectedTotal.value
        })
        successOrder.value = order
      } catch (err) {
        error.value = 'Failed to place restocking order: ' + err.message
        console.error(err)
      } finally {
        placing.value = false
      }
    }

    onMounted(loadDemand)

    return {
      t,
      loading,
      error,
      budget,
      placing,
      successOrder,
      maxBudget,
      enrichedItems,
      recommendations,
      selectedTotal,
      budgetUsedPercent,
      formatCurrency,
      formatDate,
      placeOrder
    }
  }
}
</script>

<style scoped>
/* Budget card body */
.budget-body {
  padding: 0.5rem 0;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1.25rem;
  margin-bottom: 0.75rem;
}

.budget-label {
  min-width: 160px;
  font-weight: 600;
  color: #0f172a;
  font-size: 0.938rem;
  white-space: nowrap;
}

.budget-slider {
  flex: 1;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
}

/* Progress bar */
.progress-track {
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 0.5rem;
}

.progress-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 4px;
  transition: width 0.2s ease;
}

.budget-caption {
  font-size: 0.813rem;
  color: #64748b;
}

/* Empty state cell */
.empty-state {
  text-align: center;
  color: #64748b;
  padding: 2rem 0.75rem;
  font-size: 0.875rem;
}

/* SKU code styling */
.sku {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
}

/* Table footer row */
.table-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.875rem 0.75rem 0.25rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 0.5rem;
}

.footer-summary {
  font-size: 0.875rem;
  color: #64748b;
}

/* Place Order button */
.place-order-btn {
  background: #2563eb;
  color: #ffffff;
  border: none;
  border-radius: 6px;
  padding: 0.5rem 1.25rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease, opacity 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

/* Success banner */
.success-banner {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.875rem;
}

.dismiss-btn {
  background: none;
  border: none;
  font-size: 1.25rem;
  line-height: 1;
  color: #065f46;
  cursor: pointer;
  padding: 0 0.25rem;
  opacity: 0.7;
}

.dismiss-btn:hover {
  opacity: 1;
}
</style>
