<script setup lang="ts">
import { ref } from 'vue'
import { VueScrollPicker, VueScrollPickerValue } from 'vue-scroll-picker'

const currentValue = ref<VueScrollPickerValue>(600)
</script>

<template>
  <div>
    <VueScrollPicker
      v-model="currentValue"
      placeholder="Select"
      :min="0"
      :max="1000"
      :interval="100"
    />
  </div>
</template>
