<script setup lang="ts">
import { ref } from 'vue'

const counter = ref(0)
/*
const autoIncrement = () =>
  setInterval(function () {
    counter.value++
  }, 100)
*/
const increment = () => {
  counter.value++
}
</script>

<template>
  <h1>Cookie Clicker!</h1>
  <p>{{ counter }}</p>
  <button @click="increment">Add number</button>
  <!-- <button @click="autoIncrement">Start auto-increment</button>-->
</template>
