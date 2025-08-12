<script setup lang="ts">
import { ref, watch, computed} from 'vue'
import Button from './Components/Button.vue'
import OrderItem from './Components/OrderItem.vue'

// Interfaces
interface Order {
  name: string
  time: string
  isVip: boolean
  status: 'pending' | 'processing' | 'completed'
  processedByBot?: string
  processedAt?: string
}

interface Bot {
  id: number
  name: string
  status: 'idle' | 'processing'
}

const PROCESSING_TIME = 10 // seconds

// States
const orders = ref<Order[]>([])
const bots = ref<Bot[]>([])

// For display purposes
const pendingOrders = computed(() => orders.value.filter(o => o.status === 'pending'))
const processingOrders = computed(() => orders.value.filter(o => o.status === 'processing'))
const completedOrders = computed(() => orders.value.filter(o => o.status === 'completed'))

const normalOrderCount = ref(0)
const vipOrderCount = ref(0)
const botId = ref(0)

// This function creates an order with a unique name and time
function createNewOrder(name: string, isVip: boolean): Order {
  return {
    name,
    isVip,
    time: new Date().toLocaleTimeString(),
    status: 'pending',
  }
}

// This function queues an order to be processed by a bot
function queueOrder(order: Order) {
  const pendingList = orders.value.filter(o => o.status === 'pending');

  if (order.isVip) {
    const lastVipIndex = pendingList.map(o => o.isVip).lastIndexOf(true);
    const insertIndex = lastVipIndex >= 0
        ? orders.value.findIndex(o => o === pendingList[lastVipIndex]) + 1
        : orders.value.findIndex(o => o.status === 'pending');

    if (insertIndex >= 0) {
      orders.value.splice(insertIndex, 0, order);
    } else {
      orders.value.push(order);
    }
  } else {
    orders.value.push(order);
  }
}

// This function adds a new normal order to the queue
function addNormalOrder() {
  normalOrderCount.value++
  queueOrder(createNewOrder(`O-${normalOrderCount.value}`, false))
}

// This function adds a new VIP order to the queue
function addVipOrder() {
  vipOrderCount.value++
  queueOrder(createNewOrder(`VIP-${vipOrderCount.value}`, true))
}

// This function adds a new bot to the list of bots
function addBot() {
  botId.value++
  bots.value.push({
    id: botId.value,
    name: `Bot-${botId.value}`,
    status: 'idle'
  })
}

// This function removes a bot from the list of bots
// If the bot is currently processing an order, it will disable user from removing the bot
function removeBot() {
  // If there are no bots, do nothing
  if (!bots.value.length) return

  // Get the latest bot spawned
  const latestBot = bots.value[bots.value.length - 1]

  if (latestBot.status === 'processing') {
    // If the latest bot is processing an order, drop the order back into the queue
    const orderProcessedBySelectedBot = processingOrders.value.find(o => o.processedByBot === latestBot.name)

    if (orderProcessedBySelectedBot) {
      orderProcessedBySelectedBot.processedByBot = undefined
      orderProcessedBySelectedBot.processedAt = undefined
      orderProcessedBySelectedBot.status = 'pending'
    }
  }

  // Remove the bot from the list
  bots.value = bots.value.filter(b => b !== latestBot)
}

// This function is called every second to process orders
// It will process orders until there are no more pending orders or bots available
function processOrders() {
  const idleBots = bots.value.filter(b => b.status === 'idle');
  const pendingList = orders.value.filter(o => o.status === 'pending');

  while (pendingList.length > 0 && idleBots.length > 0) {
    const bot = idleBots.shift();
    const order = pendingList.shift();

    if (!bot || !order) break;

    bot.status = 'processing';
    order.processedByBot = bot.name;
    order.status = 'processing';

    setTimeout(() => {
      // If still processed by the same bot, don't mark as completed, assuming it is failed
      if (order.processedByBot !== bot.name) return;

      order.processedAt = new Date().toLocaleTimeString();
      order.status = 'completed';

      bot.status = 'idle';
    }, PROCESSING_TIME * 1000);
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
    <p class="text-sm text-gray-700"><em>No bots added. Please add a bot to start processing orders.</em></p>
  </div>
  <div v-else class="mt-3">
    <p class="font-semibold">Bots:</p>
    <div class="flex flex-wrap gap-2">
      <div
          v-for="bot in bots"
          :key="bot.id"
          class="p-2 rounded-lg text-sm shadow-sm"
          :class="bot.status === 'idle' ? 'bg-gray-200' : 'bg-green-200'"
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
        <OrderItem :orders="processingOrders" status="processing" />
        <OrderItem :orders="pendingOrders" status="pending" />
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
      <OrderItem :orders="completedOrders" status="completed" />
    </div>
  </div>
</template>
