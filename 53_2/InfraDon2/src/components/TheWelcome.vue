<script setup lang="ts">

import { ref } from 'vue'
const counter = ref(20)

const increment = () => {
  counter.value++
}

</script>

<template>
  <h1>Hello World</h1>
  <button @click="increment">Increment</button>
  <p style="color: white; font-size: 50px;">{{ counter }}</p>
</template>
