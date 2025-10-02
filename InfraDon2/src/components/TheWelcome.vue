<script setup lang="ts">
import { ref } from 'vue';

// DATA - MODEL
const counter = ref(20)

// METHODS - CONTROLLER
const increment = () => {
  counter.value++
}

</script>

<template>
  <h1>Hello World</h1>
  <button @click="increment">Increment</button>
  <p style="color: white; font-size: 50px;">{{ counter }}</p>
</template>
