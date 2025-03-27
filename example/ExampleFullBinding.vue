<script setup lang="ts">
import { ref } from 'vue'
import { VueScrollPicker, VueScrollPickerValue } from 'vue-scroll-picker'

const markedValue = ref(865)

setInterval(() => {
  markedValue.value++
}, 200)

const currentValue = ref<VueScrollPickerValue>(0)

function formatMinutesToTime(minutes: number, format: 'HH:mm' | 'HH'): string {
  if (minutes === undefined) return ''
  const hours = Math.floor(minutes / 60)
  const mins = minutes % 60

  if (format === 'HH') {
    return `${hours.toString().padStart(2, '0')}`
  } else {
    return `${hours.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}`
  }
}
</script>

<template>
  <div>
    <VueScrollPicker
      v-model="currentValue"
      placeholder="Select"
      :min="540"
      :max="1820"
      :interval="20"
      :per-graduation-size="3"
      :marked-value="markedValue"
    >
      <template #label="{ option }">
        {{ formatMinutesToTime(option.value, 'HH') }}
      </template>
      <template #current="{ value }">
        {{ formatMinutesToTime(value, 'HH:mm') }}
      </template>
      <template #marked="{ value }">
        {{ formatMinutesToTime(value, 'HH:mm') }}
      </template>
    </VueScrollPicker>
  </div>
</template>
