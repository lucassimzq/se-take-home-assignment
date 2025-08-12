<script setup lang="ts">
import { ref, watch} from 'vue'
import Button from './Components/Button.vue'

// Interfaces
interface Order {
  name: string
  time: string
  isVip: boolean
  processedByBot?: string
  processedAt?: string
}

interface Bot {
  id: number
  name: string
  status: 'IDLE' | 'PROCESSING'
}

const PROCESSING_TIME = 10 // seconds

// States
const pendingOrders = ref<Order[]>([])
const processingOrders = ref<Order[]>([])
const completedOrders = ref<Order[]>([])
const bots = ref<Bot[]>([])

const normalOrderCount = ref(0)
const vipOrderCount = ref(0)
const botId = ref(0)

// This function creates an order with a unique name and time
function createOrder(name: string, isVip: boolean): Order {
  return {
    name,
    isVip,
    time: new Date().toLocaleTimeString()
  }
}

// This function queues an order to be processed by a bot
function queueOrder(order: Order) {
  if (order.isVip) {
    // Insert behind the last VIP order, or at start if none
    const lastVipIndex = pendingOrders.value
        .map(o => o.isVip)
        .lastIndexOf(true)
    if (lastVipIndex >= 0) {
      pendingOrders.value.splice(lastVipIndex + 1, 0, order)
    } else {
      pendingOrders.value.unshift(order)
    }
  } else {
    pendingOrders.value.push(order)
  }
}

// This function adds a new normal order to the queue
function addNormalOrder() {
  normalOrderCount.value++
  queueOrder(createOrder(`O-${normalOrderCount.value}`, false))
}

// This function adds a new VIP order to the queue
function addVipOrder() {
  vipOrderCount.value++
  queueOrder(createOrder(`VIP-${vipOrderCount.value}`, true))
}

// This function adds a new bot to the list of bots
function addBot() {
  botId.value++
  bots.value.push({
    id: botId.value,
    name: `Bot-${botId.value}`,
    status: 'IDLE'
  })
}

// This function removes a bot from the list of bots
// If the bot is currently processing an order, it will disable user from removing the bot
function removeBot() {
  // If there are no bots, do nothing
  if (!bots.value.length) return

  // Get the latest bot spawned
  const latestBot = bots.value[bots.value.length - 1]

  if (latestBot.status === 'PROCESSING') {
    // If the latest bot is processing an order, drop the order back into the queue
    const orderProcessedBySelectedBot = processingOrders.value.find(o => o.processedByBot === latestBot.name)

    if (orderProcessedBySelectedBot) {
      orderProcessedBySelectedBot.processedByBot = undefined
      queueOrder(orderProcessedBySelectedBot)
      processingOrders.value = processingOrders.value.filter(o => o !== orderProcessedBySelectedBot)
    }
  }

  // Remove the bot from the list
  bots.value = bots.value.filter(b => b !== latestBot)
}

// This function is called every second to process orders
// It will process orders until there are no more pending orders or bots available
function processOrders() {
  const idleBots = bots.value.filter(b => b.status === 'IDLE')

  while (pendingOrders.value.length > 0 && idleBots.length > 0) {
    const bot = idleBots.shift()
    const order = pendingOrders.value.shift()

    // If there's no bot or order, break the loop'
    if (!bot || !order) break

    bot.status = 'PROCESSING'
    order.processedByBot = bot.name

    processingOrders.value.push(order)

    setTimeout(() => {
      // Move from processing to completed
      const idx = processingOrders.value.indexOf(order)
      if (idx !== -1) {
        // Check if the order is being processed by a new bot
        // If so, do not move it to completed
        if (order.processedByBot !== bot.name) {
          return
        }
        processingOrders.value.splice(idx, 1)
      } else {
        // Return because if the order is not in processingOrders, meaning the bot processing it was removed
        return
      }

      order.processedAt = new Date().toLocaleTimeString()
      completedOrders.value.push(order)

      bot.status = 'IDLE'
      processOrders()
    }, PROCESSING_TIME * 1000)
  }
}

// Watch for changes in pending orders and bots to trigger order processing
watch([pendingOrders, bots], processOrders, { deep: true })
</script>

<template>
  <!-- Action Buttons -->
  <div class="mt-3 flex gap-2">
    <Button class="bg-blue-500 text-white" @click="addNormalOrder">New Normal Order</Button>
    <Button class="bg-yellow-500 text-white" @click="addVipOrder">New VIP Order</Button>
    <Button class="bg-green-500 text-white" @click="addBot">+ Bot</Button>
    <Button class="bg-red-500 text-white" @click="removeBot">– Bot</Button>
  </div>

  <!-- Bots section-->
  <div v-if="!bots.length" class="mt-3">
    <p>No bots added. Please add a bot to start processing orders.</p>
  </div>
  <div v-else class="mt-3">
    <p class="font-semibold">Bots:</p>
    <div class="flex flex-wrap gap-2">
      <div
          v-for="bot in bots"
          :key="bot.id"
          class="p-2 rounded-lg text-sm shadow-sm"
          :class="bot.status === 'IDLE' ? 'bg-gray-200' : 'bg-green-200'"
      >
        {{ bot.name }} ({{ bot.status }})
      </div>
    </div>
  </div>

  <!-- Orders section-->
  <div class="grid grid-cols-2 gap-3 mt-3">
    <!-- Pending & Processing -->
    <div class="bg-white rounded-xl shadow-lg p-4">
      <h2 class="text-xl font-semibold">Pending</h2>
      <div class="grid grid-cols-3 gap-2 mt-2 text-center text-sm font-medium">
        <span>Order</span>
        <span>Processing By</span>
        <span>Placed At</span>
      </div>

      <p v-if="!pendingOrders.length && !processingOrders.length" class="mt-2 text-gray-500">
        No pending orders.
      </p>

      <template v-else>
        <div
            v-for="order in processingOrders"
            :key="order.name"
            class="mt-2 p-2 border-2 border-orange-400 rounded-lg flex justify-between text-sm animate-pulse"
        >
          <span>{{ order.name }}</span>
          <span>{{ order.processedByBot }}</span>
          <span>{{ order.time }}</span>
        </div>
        <div
            v-for="order in pendingOrders"
            :key="order.name"
            class="mt-2 p-2 border-2 border-gray-300 rounded-lg flex justify-between text-sm"
        >
          <span>{{ order.name }}</span>
          <span>-</span>
          <span>{{ order.time }}</span>
        </div>
      </template>
    </div>

    <!-- Completed -->
    <div class="bg-white rounded-xl shadow-lg p-4">
      <h2 class="text-xl font-semibold">Completed</h2>
      <div class="grid grid-cols-3 gap-2 mt-2 text-center text-sm font-medium">
        <span>Order</span>
        <span>Processed By</span>
        <span>Completed At</span>
      </div>
      <p v-if="!completedOrders.length" class="mt-2 text-gray-500">
        No completed orders yet.
      </p>
      <div
          v-for="order in completedOrders"
          :key="order.name"
          class="mt-2 p-2 border-2 border-green-400 rounded-lg flex justify-between text-sm"
      >
        <span>{{ order.name }}</span>
        <span>{{ order.processedByBot }}</span>
        <span>{{ order.processedAt }}</span>
      </div>
    </div>
  </div>
</template>
