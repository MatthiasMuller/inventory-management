<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
          <span class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>
        <input
          type="range"
          min="0"
          max="500000"
          step="1000"
          v-model.number="budget"
          class="budget-slider"
        />
        <div class="slider-marks">
          <span>{{ currencySymbol }}0</span>
          <span>{{ currencySymbol }}500,000</span>
        </div>
        <div class="stats-grid budget-stats">
          <div class="stat-card info">
            <div class="stat-label">{{ t('restocking.totalCost') }}</div>
            <div class="stat-value">{{ currencySymbol }}{{ totalCost.toLocaleString() }}</div>
          </div>
          <div class="stat-card success">
            <div class="stat-label">{{ t('restocking.remainingBudget') }}</div>
            <div class="stat-value">{{ currencySymbol }}{{ remainingBudget.toLocaleString() }}</div>
          </div>
          <div class="stat-card warning">
            <div class="stat-label">{{ t('restocking.itemsIncluded') }}</div>
            <div class="stat-value">{{ itemsIncludedCount }}</div>
          </div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t('restocking.recommendations') }} ({{ walkedRecommendations.length }})
          </h3>
        </div>
        <div v-if="!walkedRecommendations.length" class="empty-state">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.gap') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.recommendedQty') }}</th>
                <th>{{ t('restocking.table.lineCost') }}</th>
                <th>{{ t('restocking.table.included') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="rec in walkedRecommendations"
                :key="rec.item_sku"
                :class="{ dimmed: !rec.included }"
              >
                <td><strong>{{ rec.item_sku }}</strong></td>
                <td>{{ rec.item_name }}</td>
                <td><span :class="['badge', rec.trend]">{{ rec.trend }}</span></td>
                <td>{{ rec.gap }}</td>
                <td>{{ currencySymbol }}{{ rec.unit_cost.toLocaleString() }}</td>
                <td>{{ rec.recommended_qty }}</td>
                <td><strong>{{ currencySymbol }}{{ rec.line_cost.toLocaleString() }}</strong></td>
                <td>
                  <span v-if="rec.included" class="badge success">Yes</span>
                  <span v-else class="badge" :title="t('restocking.excludedHint')">No</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div v-if="successMsg" class="success-message">{{ successMsg }}</div>
        <div v-if="errorMsg" class="error">{{ errorMsg }}</div>

        <div class="actions">
          <button
            class="place-order-btn"
            :disabled="!itemsIncludedCount || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.submitting') : t('restocking.placeOrder') }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

const TREND_RANK = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const currencySymbol = computed(() => (currentCurrency.value === 'JPY' ? '¥' : '$'))

    const forecasts = ref([])
    const budget = ref(50000)
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const successMsg = ref('')
    const errorMsg = ref('')

    const loadForecasts = async () => {
      try {
        loading.value = true
        forecasts.value = await api.getDemandForecasts()
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const candidates = computed(() => {
      return forecasts.value
        .map((f) => {
          const gap = Math.max(f.forecasted_demand - f.current_demand, 0)
          return {
            id: f.id,
            item_sku: f.item_sku,
            item_name: f.item_name,
            trend: f.trend,
            unit_cost: f.unit_cost,
            gap,
            recommended_qty: gap,
            line_cost: Math.round(gap * f.unit_cost * 100) / 100
          }
        })
        .filter((c) => c.gap > 0)
        .sort((a, b) => {
          const ra = TREND_RANK[a.trend] ?? 99
          const rb = TREND_RANK[b.trend] ?? 99
          if (ra !== rb) return ra - rb
          return b.gap - a.gap
        })
    })

    const walkedRecommendations = computed(() => {
      let remaining = budget.value
      return candidates.value.map((c) => {
        if (c.line_cost <= remaining) {
          remaining -= c.line_cost
          return { ...c, included: true }
        }
        return { ...c, included: false }
      })
    })

    const includedItems = computed(() => walkedRecommendations.value.filter((r) => r.included))
    const totalCost = computed(() =>
      Math.round(includedItems.value.reduce((sum, r) => sum + r.line_cost, 0) * 100) / 100
    )
    const remainingBudget = computed(() => Math.max(budget.value - totalCost.value, 0))
    const itemsIncludedCount = computed(() => includedItems.value.length)

    const placeOrder = async () => {
      if (!itemsIncludedCount.value) return
      submitting.value = true
      successMsg.value = ''
      errorMsg.value = ''
      try {
        const payload = {
          budget: budget.value,
          items: includedItems.value.map((r) => ({
            item_sku: r.item_sku,
            item_name: r.item_name,
            quantity: r.recommended_qty,
            unit_cost: r.unit_cost
          }))
        }
        await api.createRestockingOrder(payload)
        successMsg.value = t('restocking.submittedSuccess')
      } catch (err) {
        errorMsg.value = t('restocking.submitError') + ' (' + err.message + ')'
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      t,
      currencySymbol,
      budget,
      loading,
      error,
      submitting,
      successMsg,
      errorMsg,
      walkedRecommendations,
      totalCost,
      remainingBudget,
      itemsIncludedCount,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-slider {
  width: 100%;
  margin: 0.75rem 0 0.25rem;
  accent-color: #2563eb;
  height: 6px;
}

.slider-marks {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #64748b;
  margin-bottom: 1rem;
}

.budget-display {
  font-size: 1.75rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.budget-stats {
  margin-top: 0.5rem;
  margin-bottom: 0;
}

.dimmed {
  opacity: 0.45;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.success-message {
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  margin: 1rem 0 0;
  font-size: 0.938rem;
  font-weight: 500;
}

.actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 1.25rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  font-weight: 600;
  font-size: 0.938rem;
  padding: 0.75rem 2rem;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}
</style>
