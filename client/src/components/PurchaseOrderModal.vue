<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">
              {{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}
            </h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Backlog item summary shown in both modes -->
            <div class="item-summary">
              <div class="item-summary-info">
                <h4 class="item-name">{{ backlogItem.item_name }}</h4>
                <div class="item-sku">SKU: {{ backlogItem.item_sku }}</div>
              </div>
              <div class="shortage-badge">
                <span class="shortage-label">Shortage</span>
                <span class="shortage-value">{{ shortage }} units</span>
              </div>
            </div>

            <!-- Create mode: form -->
            <form v-if="mode === 'create'" class="po-form" @submit.prevent="submitForm">
              <div class="form-group">
                <label class="form-label" for="supplier-name">Supplier Name</label>
                <input
                  id="supplier-name"
                  v-model="form.supplier"
                  type="text"
                  class="form-input"
                  placeholder="Enter supplier name"
                  required
                />
              </div>

              <div class="form-group">
                <label class="form-label" for="quantity">Quantity to Order</label>
                <input
                  id="quantity"
                  v-model.number="form.quantity"
                  type="number"
                  class="form-input"
                  min="1"
                  required
                />
              </div>

              <div class="form-group">
                <label class="form-label" for="unit-cost">Unit Cost</label>
                <input
                  id="unit-cost"
                  v-model.number="form.unit_cost"
                  type="number"
                  class="form-input"
                  min="0"
                  step="0.01"
                  placeholder="0.00"
                  required
                />
              </div>

              <div class="form-group">
                <label class="form-label" for="delivery-date">Expected Delivery Date</label>
                <input
                  id="delivery-date"
                  v-model="form.expected_delivery"
                  type="date"
                  class="form-input"
                  required
                />
              </div>

              <div class="total-cost-preview">
                <span class="total-label">Estimated Total Cost</span>
                <span class="total-value">{{ formattedTotalCost }}</span>
              </div>
            </form>

            <!-- View mode: read-only PO details -->
            <div v-else-if="mode === 'view' && backlogItem.purchase_order" class="po-details">
              <div class="info-grid">
                <div class="info-item">
                  <div class="info-label">PO ID</div>
                  <div class="info-value po-id">{{ backlogItem.purchase_order.id }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Supplier</div>
                  <div class="info-value">{{ backlogItem.purchase_order.supplier }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Quantity</div>
                  <div class="info-value">{{ backlogItem.purchase_order.quantity }} units</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Unit Cost</div>
                  <div class="info-value">{{ formatCost(backlogItem.purchase_order.unit_cost) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Total Cost</div>
                  <div class="info-value total-highlight">{{ formatCost(backlogItem.purchase_order.total_cost) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Expected Delivery</div>
                  <div class="info-value">{{ formatDate(backlogItem.purchase_order.expected_delivery) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Status</div>
                  <div class="info-value">
                    <span class="status-badge" :class="backlogItem.purchase_order.status">
                      {{ backlogItem.purchase_order.status }}
                    </span>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close">
              {{ mode === 'create' ? 'Cancel' : 'Close' }}
            </button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              type="submit"
              @click="submitForm"
            >
              Create Purchase Order
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
  isOpen: {
    type: Boolean,
    default: false
  },
  backlogItem: {
    type: Object,
    default: null
  },
  mode: {
    type: String,
    default: 'create'
  }
})

const emit = defineEmits(['close', 'po-created'])

const form = ref({
  supplier: '',
  quantity: 0,
  unit_cost: '',
  expected_delivery: ''
})

// Pre-fill quantity when backlogItem changes or modal opens
watch(
  () => [props.isOpen, props.backlogItem],
  ([isOpen, item]) => {
    if (isOpen && item && props.mode === 'create') {
      form.value = {
        supplier: '',
        quantity: Math.max(0, item.quantity_needed - item.quantity_available),
        unit_cost: '',
        expected_delivery: ''
      }
    }
  },
  { immediate: true }
)

const shortage = computed(() => {
  if (!props.backlogItem) return 0
  return props.backlogItem.quantity_needed - props.backlogItem.quantity_available
})

const formattedTotalCost = computed(() => {
  const qty = form.value.quantity || 0
  const cost = parseFloat(form.value.unit_cost) || 0
  const total = qty * cost
  return total.toLocaleString('en-US', { style: 'currency', currency: 'USD' })
})

const formatCost = (value) => {
  if (value == null) return 'N/A'
  return Number(value).toLocaleString('en-US', { style: 'currency', currency: 'USD' })
}

const formatDate = (dateString) => {
  if (!dateString) return 'N/A'
  const date = new Date(dateString)
  if (isNaN(date.getTime())) return dateString
  return date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}

const close = () => {
  emit('close')
}

const submitForm = () => {
  if (!props.backlogItem) return

  const quantity = form.value.quantity
  const unit_cost = parseFloat(form.value.unit_cost)

  const poData = {
    id: 'PO-' + Date.now(),
    backlog_item_id: props.backlogItem.id,
    supplier: form.value.supplier,
    quantity,
    unit_cost,
    total_cost: quantity * unit_cost,
    expected_delivery: form.value.expected_delivery,
    status: 'pending',
    created_at: new Date().toISOString()
  }

  emit('po-created', poData)
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 560px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 2rem;
}

.item-summary {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  padding-bottom: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
  margin-bottom: 1.5rem;
}

.item-name {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 0.375rem 0;
}

.item-sku {
  font-size: 0.875rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

.shortage-badge {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  flex-shrink: 0;
}

.shortage-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin-bottom: 0.25rem;
}

.shortage-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #dc2626;
}

/* Form styles */
.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.form-input {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.938rem;
  color: #0f172a;
  font-family: inherit;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  outline: none;
}

.form-input:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.total-cost-preview {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 1.25rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  margin-top: 0.5rem;
}

.total-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
}

.total-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

/* View mode styles */
.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.5rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.info-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.info-value.po-id {
  font-family: 'Monaco', 'Courier New', monospace;
  color: #2563eb;
}

.info-value.total-highlight {
  font-size: 1.125rem;
  font-weight: 700;
}

.status-badge {
  display: inline-block;
  padding: 0.25rem 0.75rem;
  border-radius: 9999px;
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: capitalize;
}

.status-badge.pending {
  background: #fef3c7;
  color: #92400e;
}

.status-badge.approved {
  background: #d1fae5;
  color: #065f46;
}

.status-badge.fulfilled {
  background: #dbeafe;
  color: #1e40af;
}

.status-badge.cancelled {
  background: #fee2e2;
  color: #991b1b;
}

.modal-footer {
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.btn-secondary {
  padding: 0.625rem 1.25rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: #334155;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-secondary:hover {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #2563eb;
  border: 1px solid #2563eb;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover {
  background: #1d4ed8;
  border-color: #1d4ed8;
}

/* Modal transition animations */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>
