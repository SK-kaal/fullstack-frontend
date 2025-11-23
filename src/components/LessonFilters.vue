<script setup>
const props = defineProps({
  searchTerm: {
    type: String,
    default: '',
  },
  sortBy: {
    type: String,
    default: 'subject',
  },
  sortDirection: {
    type: String,
    default: 'asc',
  },
  sortOptions: {
    type: Array,
    default: () => [],
  },
});

const emit = defineEmits(['update:searchTerm', 'update:sortBy', 'update:sortDirection']);
</script>

<template>
  <section class="card shadow-sm p-4 mb-4">
    <div class="row g-3 align-items-end">
      <div class="col-md-6">
        <label class="form-label">Search lessons</label>
        <input
          :value="searchTerm"
          @input="emit('update:searchTerm', $event.target.value)"
          type="text"
          class="form-control"
          placeholder="Type to search across subject, location, price or spaces"
        />
      </div>
      <div class="col-md-3">
        <label class="form-label">Sort by</label>
        <select :value="sortBy" @change="emit('update:sortBy', $event.target.value)" class="form-select">
          <option v-for="option in sortOptions" :key="option.value" :value="option.value">
            {{ option.label }}
          </option>
        </select>
      </div>
      <div class="col-md-3">
        <label class="form-label">Order</label>
        <select
          :value="sortDirection"
          @change="emit('update:sortDirection', $event.target.value)"
          class="form-select"
        >
          <option value="asc">Ascending</option>
          <option value="desc">Descending</option>
        </select>
      </div>
    </div>
  </section>
</template>

