<script setup>
const props = defineProps({
  cart: {
    type: Array,
    default: () => [],
  },
  cartCount: Number,
  cartTotal: Number,
  checkoutForm: {
    type: Object,
    default: () => ({ name: '', phone: '' }),
  },
  isNameValid: Boolean,
  isPhoneValid: Boolean,
  canCheckout: Boolean,
  submitting: Boolean,
  formatPrice: {
    type: Function,
    default: (value) => value,
  },
  showCheckoutToast: Boolean,
});

const emit = defineEmits(['remove', 'checkout']);
</script>

<template>
  <section class="card shadow-sm p-4">
    <div class="d-flex justify-content-between align-items-center mb-3">
      <h2 class="h4 mb-0">Shopping cart</h2>
      <span class="badge text-bg-secondary">{{ cartCount }} items</span>
    </div>

    <div v-if="cart.length === 0" class="alert alert-info">
      Your cart is empty. Head back to the lessons list to add some activities.
    </div>

    <div v-else>
      <ul class="list-group mb-3 cart-list">
        <li
          v-for="item in cart"
          :key="item.lessonId"
          class="list-group-item d-flex justify-content-between align-items-center"
        >
          <div>
            <h6 class="mb-1">{{ item.subject }}</h6>
            <small class="text-muted">Quantity: {{ item.quantity }} × {{ formatPrice(item.price) }}</small>
          </div>
          <div class="d-flex align-items-center">
            <strong class="me-3">{{ formatPrice(item.price * item.quantity) }}</strong>
            <button class="btn btn-outline-danger btn-sm" @click="emit('remove', item.lessonId)">
              <i class="fa-solid fa-trash-can"></i>
            </button>
          </div>
        </li>
      </ul>

      <div class="d-flex justify-content-between align-items-center mb-4">
        <strong>Total</strong>
        <span class="fs-5 text-primary fw-bold">{{ formatPrice(cartTotal) }}</span>
      </div>

      <form class="row g-3" @submit.prevent="emit('checkout')">
        <div class="col-md-6">
          <label class="form-label">Name</label>
          <input
            v-model="checkoutForm.name"
            type="text"
            class="form-control"
            :class="{ 'is-invalid': checkoutForm.name && !isNameValid }"
            placeholder="e.g. Taylor Green"
            required
          />
          <div class="invalid-feedback">Letters and spaces only.</div>
        </div>
        <div class="col-md-6">
          <label class="form-label">Phone</label>
          <input
            v-model="checkoutForm.phone"
            type="tel"
            class="form-control"
            :class="{ 'is-invalid': checkoutForm.phone && !isPhoneValid }"
            placeholder="07123456789"
            required
          />
          <div class="invalid-feedback">Digits only (minimum 6).</div>
        </div>
        <div class="col-12">
          <button class="btn btn-success" type="submit" :disabled="!canCheckout">
            <span v-if="submitting" class="spinner-border spinner-border-sm me-2"></span>
            Checkout
          </button>
        </div>
      </form>
      <p v-if="showCheckoutToast" class="mt-3 text-success fw-semibold">
        Your order is confirmed! We'll text you a confirmation shortly.
      </p>
    </div>
  </section>
</template>

