<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Recommend items to restock based on demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-body">
          <div class="budget-label">Available Budget: {{ currencySymbol }}{{ budget.toLocaleString() }}</div>
          <div class="slider-wrapper">
            <span class="slider-min">{{ currencySymbol }}0</span>
            <input
              type="range"
              class="budget-slider"
              min="0"
              max="500000"
              step="1000"
              v-model.number="budget"
              @input="onBudgetChange"
            />
            <span class="slider-max">{{ currencySymbol }}500,000</span>
          </div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <div class="card-header-row">
            <h3 class="card-title">
              Recommended Items
              <span class="count-info">({{ fittingItems.length }} fit within budget / {{ recommendedItems.length }} total)</span>
            </h3>
            <button
              class="btn-primary"
              :disabled="fittingItems.length === 0 || submitting"
              @click="placeOrder"
            >
              {{ submitting ? 'Submitting...' : 'Place Order' }}
            </button>
          </div>
        </div>

        <div v-if="successMessage" class="success-banner">
          {{ successMessage }}
        </div>

        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>Trend</th>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Demand Gap</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
                <th>Fits?</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendedItems"
                :key="item.id"
                :class="{ 'over-budget': !item.fits }"
              >
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td><code>{{ item.sku }}</code></td>
                <td>{{ item.name }}</td>
                <td>{{ item.gap.toLocaleString() }}</td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
                <td>{{ currencySymbol }}{{ item.item_total.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
                <td>
                  <span v-if="item.fits" class="badge success">Fits</span>
                  <span v-else class="over-budget-label">Over budget</span>
                </td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="totals-row">
                <td colspan="5"><strong>Total</strong></td>
                <td><strong>{{ currencySymbol }}{{ totalCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></td>
                <td class="budget-note">of {{ currencySymbol }}{{ budget.toLocaleString() }} budget</td>
              </tr>
            </tfoot>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

const TREND_ORDER = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const loading = ref(true)
    const error = ref(null)
    const demandForecasts = ref([])
    const inventoryItems = ref([])
    const budget = ref(50000)
    const submitting = ref(false)
    const successMessage = ref('')

    // Build name -> unit_cost lookup and compute average unit cost
    const unitCostLookup = computed(() => {
      const lookup = {}
      for (const item of inventoryItems.value) {
        lookup[item.name.toLowerCase()] = item.unit_cost
      }
      return lookup
    })

    const averageUnitCost = computed(() => {
      if (inventoryItems.value.length === 0) return 0
      const total = inventoryItems.value.reduce((sum, item) => sum + item.unit_cost, 0)
      return total / inventoryItems.value.length
    })

    // All recommended items (gap > 0), sorted by trend
    const recommendedItems = computed(() => {
      const items = demandForecasts.value
        .map(item => {
          const gap = item.forecasted_demand - item.current_demand
          if (gap <= 0) return null

          const nameLower = item.item_name.toLowerCase()
          const unit_cost = unitCostLookup.value[nameLower] !== undefined
            ? unitCostLookup.value[nameLower]
            : averageUnitCost.value

          return {
            id: item.id,
            sku: item.item_sku,
            name: item.item_name,
            trend: item.trend,
            gap,
            unit_cost,
            item_total: gap * unit_cost,
            fits: false
          }
        })
        .filter(Boolean)
        .sort((a, b) => (TREND_ORDER[a.trend] ?? 99) - (TREND_ORDER[b.trend] ?? 99))

      // Greedily mark fits
      let cumulative = 0
      for (const item of items) {
        if (cumulative + item.item_total <= budget.value) {
          item.fits = true
          cumulative += item.item_total
        } else {
          item.fits = false
        }
      }

      return items
    })

    const fittingItems = computed(() => {
      return recommendedItems.value.filter(item => item.fits)
    })

    const totalCost = computed(() => {
      return fittingItems.value.reduce((sum, item) => sum + item.item_total, 0)
    })

    const onBudgetChange = () => {
      successMessage.value = ''
    }

    const placeOrder = async () => {
      if (fittingItems.value.length === 0 || submitting.value) return
      submitting.value = true
      successMessage.value = ''
      try {
        const result = await api.createRestockingOrder({
          items: fittingItems.value.map(i => ({
            sku: i.sku,
            name: i.name,
            quantity: i.gap,
            unit_cost: i.unit_cost
          }))
        })
        const deliveryDate = result.expected_delivery
          ? new Date(result.expected_delivery).toLocaleDateString('en-US', {
              year: 'numeric', month: 'short', day: 'numeric'
            })
          : 'TBD'
        successMessage.value = `Order ${result.order_number} submitted! Expected delivery: ${deliveryDate}`
      } catch (err) {
        error.value = 'Failed to submit order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        demandForecasts.value = forecasts
        inventoryItems.value = inventory
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      budget,
      submitting,
      successMessage,
      recommendedItems,
      fittingItems,
      totalCost,
      currencySymbol,
      onBudgetChange,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-card .budget-body {
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.budget-label {
  font-size: 1rem;
  font-weight: 600;
  color: #0f172a;
}

.slider-wrapper {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.slider-min,
.slider-max {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
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
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
  transition: background 0.15s;
}

.budget-slider::-webkit-slider-thumb:hover {
  background: #1d4ed8;
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: none;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.card-header-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
}

.count-info {
  font-size: 0.875rem;
  font-weight: 400;
  color: #64748b;
  margin-left: 0.5rem;
}

.btn-primary {
  padding: 0.5rem 1.25rem;
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.15s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-banner {
  margin: 0 1.5rem 1rem;
  padding: 0.75rem 1rem;
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  border-radius: 6px;
  color: #166534;
  font-size: 0.875rem;
  font-weight: 500;
}

tr.over-budget {
  opacity: 0.4;
}

.over-budget-label {
  font-size: 0.813rem;
  color: #94a3b8;
}

.totals-row td {
  border-top: 2px solid #e2e8f0;
  padding-top: 0.75rem;
  background: #f8fafc;
}

.budget-note {
  font-size: 0.813rem;
  color: #64748b;
}

code {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-size: 0.813rem;
  color: #475569;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 3px;
}
</style>
