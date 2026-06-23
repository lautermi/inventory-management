<template>
  <div class="restocking">
    <!-- Page header -->
    <div class="page-header">
      <h2>{{ t("nav.restocking") }}</h2>
      <p>Recommended restocking based on demand forecasts</p>
    </div>

    <!-- Budget card -->
    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Budget</h3>
        <span class="budget-display">${{ budget.toLocaleString() }}</span>
      </div>
      <input
        type="range"
        min="0"
        max="50000"
        step="500"
        v-model.number="budget"
        class="budget-slider"
      />
      <div class="budget-meta">
        <span>$0</span>
        <span class="budget-selected-total"
          >Selected: ${{ selectedTotal.toLocaleString() }}</span
        >
        <span>$50,000</span>
      </div>
    </div>

    <!-- Loading / error states -->
    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else class="recommendations-section">
      <!-- Section header with Place Order button -->
      <div class="section-header">
        <h3 class="card-title">
          Recommended Items ({{ withinBudgetItems.length }} within budget)
        </h3>
        <button
          v-if="withinBudgetItems.length > 0"
          @click="placeOrder"
          :disabled="placing"
          class="btn-place-order"
        >
          {{ placing ? "Placing Order..." : "Place Order" }}
        </button>
      </div>

      <!-- Success banner -->
      <div v-if="orderSuccess" class="order-success-banner">
        Order placed successfully! Check the Orders tab for status and delivery
        date.
      </div>

      <!-- Recommendations grid -->
      <div class="recommendations-grid">
        <div
          v-for="item in recommendations"
          :key="item.id"
          :class="[
            'recommendation-card',
            {
              'within-budget': isWithinBudget(item),
              'over-budget': !isWithinBudget(item),
            },
          ]"
        >
          <div class="rec-header">
            <span class="rec-sku">{{ item.item_sku }}</span>
            <span
              :class="[
                'badge',
                item.trend === 'increasing'
                  ? 'success'
                  : item.trend === 'stable'
                    ? 'info'
                    : 'warning',
              ]"
            >
              {{ item.trend }}
            </span>
          </div>
          <div class="rec-name">{{ item.item_name }}</div>
          <div class="rec-stats">
            <div class="rec-stat">
              <span class="rec-stat-label">Current</span>
              <span class="rec-stat-value">{{ item.current_demand }}</span>
            </div>
            <div class="rec-stat">
              <span class="rec-stat-label">Forecasted</span>
              <span class="rec-stat-value"
                ><strong>{{ item.forecasted_demand }}</strong></span
              >
            </div>
            <div class="rec-stat">
              <span class="rec-stat-label">Unit Cost</span>
              <span class="rec-stat-value">
                ${{ item.unit_cost.toFixed(2) }}
                <span v-if="item.price_source === 'estimated'" class="est-badge"
                  >est.</span
                >
              </span>
            </div>
            <div class="rec-stat">
              <span class="rec-stat-label">Total Cost</span>
              <span class="rec-stat-value cost-highlight"
                >${{ item.total_cost.toLocaleString() }}</span
              >
            </div>
          </div>
          <div v-if="!isWithinBudget(item)" class="over-budget-label">
            Over budget
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from "vue";
import { api } from "../api";
import { useI18n } from "../composables/useI18n";

export default {
  name: "Restocking",
  setup() {
    const { t } = useI18n();
    const budget = ref(25000);
    const recommendations = ref([]);
    const loading = ref(true);
    const error = ref(null);
    const placing = ref(false);
    const orderSuccess = ref(false);

    const loadRecommendations = async () => {
      try {
        loading.value = true;
        error.value = null;
        recommendations.value = await api.getRestockingRecommendations();
      } catch (err) {
        error.value = "Failed to load recommendations: " + err.message;
      } finally {
        loading.value = false;
      }
    };

    // Greedy selection: iterate recommendations in priority order,
    // accumulate running total, include item only if it fits within budget
    const withinBudgetItems = computed(() => {
      let running = 0;
      return recommendations.value.filter((item) => {
        if (running + item.total_cost <= budget.value) {
          running += item.total_cost;
          return true;
        }
        return false;
      });
    });

    const selectedTotal = computed(() =>
      withinBudgetItems.value.reduce((sum, item) => sum + item.total_cost, 0),
    );

    // Check membership by id rather than reference equality (array is recomputed)
    const isWithinBudget = (item) =>
      withinBudgetItems.value.some((i) => i.id === item.id);

    const placeOrder = async () => {
      placing.value = true;
      orderSuccess.value = false;
      try {
        const items = withinBudgetItems.value.map((item) => ({
          sku: item.item_sku,
          name: item.item_name,
          quantity: item.restock_quantity,
          unit_price: item.unit_cost,
        }));
        await api.createRestockingOrder({
          items,
          total_value: selectedTotal.value,
        });
        orderSuccess.value = true;
      } catch (err) {
        error.value = "Failed to place order: " + err.message;
      } finally {
        placing.value = false;
      }
    };

    onMounted(loadRecommendations);

    return {
      t,
      budget,
      recommendations,
      loading,
      error,
      placing,
      orderSuccess,
      withinBudgetItems,
      selectedTotal,
      isWithinBudget,
      placeOrder,
    };
  },
};
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-card .card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #3b82f6;
  cursor: pointer;
}

.budget-meta {
  display: flex;
  justify-content: space-between;
  font-size: 0.875rem;
  color: #64748b;
  margin-top: 0.5rem;
}

.budget-selected-total {
  font-weight: 600;
  color: #0f172a;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.btn-place-order {
  background: #3b82f6;
  color: white;
  border: none;
  padding: 0.625rem 1.25rem;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  font-size: 0.875rem;
  transition: background 0.2s;
}

.btn-place-order:hover:not(:disabled) {
  background: #2563eb;
}

.btn-place-order:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.order-success-banner {
  background: #f0fdf4;
  border: 1px solid #86efac;
  color: #166534;
  padding: 0.875rem 1rem;
  border-radius: 6px;
  margin-bottom: 1rem;
  font-weight: 500;
}

.recommendations-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 1rem;
}

.recommendation-card {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1rem;
  transition:
    opacity 0.2s,
    border-color 0.2s;
}

.recommendation-card.within-budget {
  border-color: #3b82f6;
  box-shadow: 0 0 0 1px #3b82f6;
}

.recommendation-card.over-budget {
  opacity: 0.5;
}

.rec-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
}

.rec-sku {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  font-family: monospace;
}

.rec-name {
  font-weight: 600;
  color: #0f172a;
  margin-bottom: 0.75rem;
  font-size: 0.9375rem;
}

.rec-stats {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.5rem;
}

.rec-stat {
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
}

.rec-stat-label {
  font-size: 0.75rem;
  color: #64748b;
}

.rec-stat-value {
  font-size: 0.875rem;
  color: #0f172a;
}

.cost-highlight {
  font-weight: 700;
  color: #0f172a;
}

.est-badge {
  font-size: 0.7rem;
  color: #94a3b8;
  margin-left: 0.25rem;
}

.over-budget-label {
  margin-top: 0.75rem;
  font-size: 0.75rem;
  color: #ef4444;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}
</style>
