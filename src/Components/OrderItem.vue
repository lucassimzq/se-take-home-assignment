<template>
  <div
      v-for="order in orders"
      :key="order.name"
      class="mt-2 p-2 border-2 rounded-lg flex justify-between text-sm"
      :class="classStyle[status]"
  >
    <span>{{ order.name }}</span>
    <span>{{ order.processedByBot }}</span>
    <span v-if="status == 'completed'">{{ order.processedAt }}</span>
    <span v-else>{{ order.time }}</span>
  </div>
</template>

<script setup lang="ts">
import {ref} from "vue";

interface Order {
  name: string
  time: string
  isVip: boolean
  processedByBot?: string
  processedAt?: string
}

const classStyle = ref({
  pending: 'border-gray-300',
  processing: 'border-orange-400 animate-pulse',
  completed: 'border-green-400',
})

defineProps<{
  orders: Order[];
  status: 'pending' | 'processing' | 'completed';
}>();
</script>
