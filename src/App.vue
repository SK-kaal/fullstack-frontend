<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue';

const apiBaseUrl = (import.meta.env.VITE_API_BASE_URL || 'http://localhost:4000').replace(/\/$/, '');

const lessons = ref([]);
const lessonStore = reactive({});
const cart = ref([]);
const isCartVisible = ref(false);
const loading = ref(false);
const errorMessage = ref('');
const showCheckoutToast = ref(false);
const checkoutMessage = ref('');
const checkoutStatus = ref('success');
const submitting = ref(false);
let toastTimeout;
let spinnerTimeout;
const MIN_SPINNER_DURATION = 1000;
const searchTerm = ref('');
const sortBy = ref('subject');
const sortDirection = ref('asc');
const checkoutForm = reactive({ name: '', phone: '' });
let searchDebounce;

const sortOptions = [
  { value: 'subject', label: 'Subject' },
  { value: 'location', label: 'Location' },
  { value: 'price', label: 'Price' },
  { value: 'spaces', label: 'Spaces' },
];

const formatCurrency = (value) =>
  new Intl.NumberFormat('en-GB', { style: 'currency', currency: 'GBP' }).format(value);

const resolveImage = (path) => {
  if (!path) return 'https://via.placeholder.com/120x120?text=Class';
  if (path.startsWith('http')) return path;
  return `${apiBaseUrl}${path}`;
};

const getLessonById = (id) => lessonStore[id];

const upsertLessons = (list) => {
  const refs = list.map((lesson) => {
    if (lessonStore[lesson._id]) {
      Object.assign(lessonStore[lesson._id], lesson);
      return lessonStore[lesson._id];
    }
    lessonStore[lesson._id] = { ...lesson };
    return lessonStore[lesson._id];
  });
  lessons.value = refs;
};

const sortedLessons = computed(() => {
  const direction = sortDirection.value === 'asc' ? 1 : -1;
  return [...lessons.value].sort((a, b) => {
    const field = sortBy.value;
    const left = typeof a[field] === 'string' ? a[field].toLowerCase() : Number(a[field]);
    const right = typeof b[field] === 'string' ? b[field].toLowerCase() : Number(b[field]);
    if (left < right) return -1 * direction;
    if (left > right) return 1 * direction;
    return 0;
  });
});

const cartCount = computed(() => cart.value.reduce((sum, item) => sum + item.quantity, 0));
const cartTotal = computed(() => cart.value.reduce((sum, item) => sum + item.quantity * item.price, 0));
const isNameValid = computed(() => /^[a-zA-Z\s]+$/.test(checkoutForm.name.trim()));
const isPhoneValid = computed(() => /^\d{6,}$/.test(checkoutForm.phone.trim()));
const canCheckout = computed(
  () => cart.value.length > 0 && isNameValid.value && isPhoneValid.value && !submitting.value
);
const cartButtonDisabled = computed(() => cart.value.length === 0 && !isCartVisible.value);
const cartButtonLabel = computed(() => (isCartVisible.value ? 'Back to lessons' : `Cart (${cartCount.value})`));
const checkoutStatusClass = computed(() =>
  checkoutStatus.value === 'success' ? 'text-success' : 'text-danger'
);

const requestJson = async (path, options = {}) => {
  const url = `${apiBaseUrl}${path}`;
  let response;
  try {
    response = await fetch(url, options);
  } catch (error) {
    throw new Error('Network error. Please try again.');
  }

  let data = null;
  const isJson = response.headers.get('content-type')?.includes('application/json');
  if (isJson) {
    data = await response.json();
  }

  if (!response.ok) {
    throw new Error(data?.error || 'Request failed');
  }
  return data;
};

const loadLessons = async (query = '') => {
  loading.value = true;
  try {
    const endpoint = query ? `/search?q=${encodeURIComponent(query)}` : '/lessons';
    const data = await requestJson(endpoint);
    upsertLessons(data);
    errorMessage.value = '';
  } catch (error) {
    errorMessage.value = error.message || 'Unable to load lessons.';
  } finally {
    loading.value = false;
  }
};

const addToCart = (lesson) => {
  if (!lesson || lesson.spaces < 1) return;
  lesson.spaces -= 1;
  const existing = cart.value.find((item) => item.lessonId === lesson._id);
  if (existing) {
    existing.quantity += 1;
  } else {
    cart.value.push({
      lessonId: lesson._id,
      subject: lesson.subject,
      price: lesson.price,
      quantity: 1,
    });
  }
};

const removeFromCart = (lessonId) => {
  const cartItem = cart.value.find((item) => item.lessonId === lessonId);
  if (!cartItem) return;
  cartItem.quantity -= 1;
  const lesson = getLessonById(lessonId);
  if (lesson) {
    lesson.spaces += 1;
  }
  if (cartItem.quantity === 0) {
    cart.value = cart.value.filter((item) => item.lessonId !== lessonId);
  }
};

const toggleCartView = () => {
  if (!isCartVisible.value && cart.value.length === 0) {
    return;
  }
  isCartVisible.value = !isCartVisible.value;
};

const submitOrder = async () => {
  if (!canCheckout.value) return;
  submitting.value = true;
  clearTimeout(toastTimeout);
  showCheckoutToast.value = true;
  toastTimeout = setTimeout(() => {
    showCheckoutToast.value = false;
  }, 3000);
  checkoutMessage.value = '';
  checkoutStatus.value = 'success';
  const spinnerStart = performance.now();
  try {
    const orderPayload = {
      name: checkoutForm.name.trim(),
      phone: checkoutForm.phone.trim(),
      items: cart.value.map((item) => ({ classId: item.lessonId, quantity: item.quantity })),
    };

    await requestJson('/orders', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(orderPayload),
    });

    await Promise.all(
      cart.value.map((item) => {
        const lesson = getLessonById(item.lessonId);
        return requestJson(`/lessons/${item.lessonId}`, {
          method: 'PUT',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ spaces: lesson?.spaces ?? 0 }),
        });
      })
    );

    checkoutStatus.value = 'success';
    checkoutMessage.value = 'Order submitted successfully!';
    cart.value = [];
    checkoutForm.name = '';
    checkoutForm.phone = '';
    isCartVisible.value = false;
    await loadLessons(searchTerm.value);
  } catch (error) {
    checkoutStatus.value = 'error';
    checkoutMessage.value = error.message || 'Unable to submit order.';
  } finally {
    const elapsed = performance.now() - spinnerStart;
    const delay = Math.max(0, MIN_SPINNER_DURATION - elapsed);
    clearTimeout(spinnerTimeout);
    spinnerTimeout = setTimeout(() => {
      submitting.value = false;
    }, delay);
  }
};

watch(
  () => cart.value.length,
  (count) => {
    if (count === 0 && isCartVisible.value) {
      isCartVisible.value = false;
    }
  }
);

watch(
  () => searchTerm.value,
  (value) => {
    clearTimeout(searchDebounce);
    searchDebounce = setTimeout(() => {
      loadLessons(value.trim());
    }, 400);
  }
);

onMounted(() => {
  loadLessons();
});

onBeforeUnmount(() => {
  clearTimeout(searchDebounce);
  clearTimeout(spinnerTimeout);
  clearTimeout(toastTimeout);
});
</script>

<template>
  <div class="container py-4">
    <header class="app-header p-4 mb-4">
      <div class="d-flex flex-column flex-md-row justify-content-between align-items-md-center gap-3">
        <div>
          <p class="text-light text-uppercase mb-1">After School Hub</p>
          <h1 class="h3 fw-bold mb-0">Discover classes & book activities</h1>
        </div>
        <button
          class="btn btn-light btn-icon"
          :disabled="cartButtonDisabled"
          @click="toggleCartView"
        >
          <i class="fa-solid" :class="isCartVisible ? 'fa-list-ul' : 'fa-cart-shopping'"></i>
          {{ cartButtonLabel }}
        </button>
      </div>
    </header>

    <section class="card shadow-sm p-4 mb-4">
      <div class="row g-3 align-items-end">
        <div class="col-md-6">
          <label class="form-label">Search lessons</label>
          <input
            v-model="searchTerm"
            type="text"
            class="form-control"
            placeholder="Type to search across subject, location, price or spaces"
          />
        </div>
        <div class="col-md-3">
          <label class="form-label">Sort by</label>
          <select v-model="sortBy" class="form-select">
            <option v-for="option in sortOptions" :key="option.value" :value="option.value">
              {{ option.label }}
            </option>
          </select>
        </div>
        <div class="col-md-3">
          <label class="form-label">Order</label>
          <select v-model="sortDirection" class="form-select">
            <option value="asc">Ascending</option>
            <option value="desc">Descending</option>
          </select>
        </div>
      </div>
    </section>

    <section v-if="!isCartVisible">
      <div v-if="errorMessage" class="alert alert-danger">{{ errorMessage }}</div>
      <div v-else-if="loading" class="text-center py-5">
        <div class="spinner-border text-primary mb-3"></div>
        <p class="mb-0">Fetching the latest lessons...</p>
      </div>
      <div v-else class="row g-4">
        <div v-for="lesson in sortedLessons" :key="lesson._id" class="col-md-6 col-lg-4">
          <div class="card lesson-card h-100">
            <img :src="resolveImage(lesson.image)" :alt="lesson.subject" />
            <div class="card-body d-flex flex-column">
              <div class="d-flex justify-content-between align-items-start mb-2">
                <h5 class="card-title mb-0">{{ lesson.subject }}</h5>
                <span class="badge text-bg-primary">{{ lesson.day }}</span>
              </div>
              <p class="text-muted mb-1">
                <i class="fa-solid fa-location-dot me-2"></i>{{ lesson.location }}
              </p>
              <p class="mb-1 fw-semibold">
                <i class="fa-solid fa-sterling-sign me-2"></i>{{ formatCurrency(lesson.price) }}
              </p>
              <p :class="['mb-3 fw-semibold', lesson.spaces === 0 ? 'text-danger' : 'text-success']">
                {{ lesson.spaces }} spaces left
              </p>
              <p class="flex-grow-1">{{ lesson.description }}</p>
              <button
                class="btn btn-primary mt-3"
                :disabled="lesson.spaces === 0"
                @click="addToCart(lesson)"
              >
                <i class="fa-solid fa-cart-plus me-2"></i>Add to cart
              </button>
            </div>
          </div>
        </div>
      </div>
    </section>

    <section v-else class="card shadow-sm p-4">
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
              <small class="text-muted">Quantity: {{ item.quantity }} × {{ formatCurrency(item.price) }}</small>
            </div>
            <div class="d-flex align-items-center">
              <strong class="me-3">{{ formatCurrency(item.price * item.quantity) }}</strong>
              <button class="btn btn-outline-danger btn-sm" @click="removeFromCart(item.lessonId)">
                <i class="fa-solid fa-trash-can"></i>
              </button>
            </div>
          </li>
        </ul>

        <div class="d-flex justify-content-between align-items-center mb-4">
          <strong>Total</strong>
          <span class="fs-5 text-primary fw-bold">{{ formatCurrency(cartTotal) }}</span>
        </div>

        <form class="row g-3" @submit.prevent="submitOrder">
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
  </div>
</template>
