<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue';
import LessonList from './components/LessonList.vue';
import CartSection from './components/CartSection.vue';
import AppHeader from './components/AppHeader.vue';
import LessonFilters from './components/LessonFilters.vue';

const apiBaseUrl = (import.meta.env.VITE_API_BASE_URL || 'http://localhost:4000').replace(/\/$/, '');

const lessons = ref([]);
const lessonStore = reactive({});
const cart = ref([]);
const isCartVisible = ref(false);
const loading = ref(false);
const errorMessage = ref('');
const showCheckoutToast = ref(false);
const submitting = ref(false);
const searchTerm = ref('');
const sortBy = ref('subject');
const sortDirection = ref('asc');
const checkoutForm = reactive({ name: '', phone: '' });

const MIN_SPINNER_DURATION = 1000;
const TOAST_DURATION = 3000;

let toastTimeout;
let spinnerTimeout;
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

const showToast = () => {
  clearTimeout(toastTimeout);
  showCheckoutToast.value = true;
  toastTimeout = setTimeout(() => {
    showCheckoutToast.value = false;
  }, TOAST_DURATION);
};

const finishWithSpinnerDelay = (startedAt) => {
  const elapsed = performance.now() - startedAt;
  const delay = Math.max(0, MIN_SPINNER_DURATION - elapsed);
  clearTimeout(spinnerTimeout);
  spinnerTimeout = setTimeout(() => {
    submitting.value = false;
  }, delay);
};

const resetCheckoutForm = () => {
  cart.value = [];
  checkoutForm.name = '';
  checkoutForm.phone = '';
  isCartVisible.value = false;
};

const submitOrder = async () => {
  if (!canCheckout.value) return;
  submitting.value = true;
  showToast();
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

    resetCheckoutForm();
    await loadLessons(searchTerm.value);
  } catch (error) {
    console.error('Checkout failed', error);
  } finally {
    finishWithSpinnerDelay(spinnerStart);
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
    <AppHeader>
      <template #actions>
        <button class="btn btn-light btn-icon" :disabled="cartButtonDisabled" @click="toggleCartView">
          <i class="fa-solid" :class="isCartVisible ? 'fa-list-ul' : 'fa-cart-shopping'"></i>
          {{ cartButtonLabel }}
        </button>
      </template>
    </AppHeader>

    <LessonFilters
      :search-term="searchTerm"
      :sort-by="sortBy"
      :sort-direction="sortDirection"
      :sort-options="sortOptions"
      @update:searchTerm="(value) => (searchTerm = value)"
      @update:sortBy="(value) => (sortBy = value)"
      @update:sortDirection="(value) => (sortDirection = value)"
    />

    <LessonList
      v-if="!isCartVisible"
      :lessons="sortedLessons"
      :loading="loading"
      :errorMessage="errorMessage"
      :formatPrice="formatCurrency"
      :resolveImage="resolveImage"
      @add-to-cart="addToCart"
    />

    <CartSection
      v-else
      :cart="cart"
      :cart-count="cartCount"
      :cart-total="cartTotal"
      :checkout-form="checkoutForm"
      :is-name-valid="isNameValid"
      :is-phone-valid="isPhoneValid"
      :can-checkout="canCheckout"
      :submitting="submitting"
      :show-checkout-toast="showCheckoutToast"
      :format-price="formatCurrency"
      @remove="removeFromCart"
      @checkout="submitOrder"
    />
  </div>
</template>
