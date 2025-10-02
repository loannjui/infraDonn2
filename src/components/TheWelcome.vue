<script setup lang="ts">
import { ref } from 'vue'

const counter = ref(0)
let interval: number | null = null
function startCounter() {
  if (interval) return

  interval = setInterval(() => {
    counter.value++
    if (counter.value >= 9999) {
      clearInterval(interval!)
      interval = null
    }
  }, 1000)
}
const increment = () => {
  counter.value++
}
</script>

<template>
  <h1>Cookie Clicker!</h1>
  <p>{{ counter }}</p>
  <button @click="increment">Add number</button>
  <button @click="startCounter">Start auto-increment</button>
</template>
