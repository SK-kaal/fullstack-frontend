<script setup>
const props = defineProps({
  lessons: {
    type: Array,
    default: () => [],
  },
  loading: Boolean,
  errorMessage: String,
  formatPrice: {
    type: Function,
    default: (value) => value,
  },
  resolveImage: {
    type: Function,
    default: (path) => path,
  },
});

const emit = defineEmits(['add-to-cart']);
</script>

<template>
  <section>
    <div v-if="props.errorMessage" class="alert alert-danger">{{ props.errorMessage }}</div>
    <div v-else-if="props.loading" class="text-center py-5">
      <div class="spinner-border text-primary mb-3"></div>
      <p class="mb-0">Fetching the latest lessons...</p>
    </div>
    <div v-else class="row g-4">
      <div v-for="lesson in props.lessons" :key="lesson._id" class="col-md-6 col-lg-4">
        <div class="card lesson-card h-100">
          <img :src="props.resolveImage(lesson.image)" :alt="lesson.subject" />
          <div class="card-body d-flex flex-column">
            <div class="d-flex justify-content-between align-items-start mb-2">
              <h5 class="card-title mb-0">{{ lesson.subject }}</h5>
              <span class="badge text-bg-primary">{{ lesson.day }}</span>
            </div>
            <p class="text-muted mb-1">
              <i class="fa-solid fa-location-dot me-2"></i>{{ lesson.location }}
            </p>
            <p class="mb-1 fw-semibold">
              <i class="fa-solid fa-sterling-sign me-2"></i>{{ props.formatPrice(lesson.price) }}
            </p>
            <p :class="['mb-3 fw-semibold', lesson.spaces === 0 ? 'text-danger' : 'text-success']">
              {{ lesson.spaces }} spaces left
            </p>
            <p class="flex-grow-1">{{ lesson.description }}</p>
            <button class="btn btn-primary mt-3" :disabled="lesson.spaces === 0" @click="emit('add-to-cart', lesson)">
              <i class="fa-solid fa-cart-plus me-2"></i>Add to cart
            </button>
          </div>
        </div>
      </div>
    </div>
  </section>
</template>

