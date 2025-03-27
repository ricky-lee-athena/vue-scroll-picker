<script
  setup
  lang="ts"
  generic="T extends ScrollPickerOption = ScrollPickerOption"
>
import {
  computed,
  nextTick,
  onBeforeUnmount,
  onMounted,
  ref,
  shallowRef,
  watch,
} from 'vue'

export type ScrollPickerValue = number
export interface ScrollPickerOption {
  value: ScrollPickerValue
  disabled?: boolean
}

export type ScrollPickerOptionable<T> = ScrollPickerValue | T

type GestureState = [scrollX: number, clientX: number, isDragging: boolean]

const EVENT_OPTIONS: AddEventListenerOptions = { passive: false }

const emit = defineEmits<{
  (event: 'update:modelValue', value: ScrollPickerValue | undefined): void
  (event: 'start'): void
  (event: 'move', value: ScrollPickerValue | undefined): void
  (event: 'end', value: ScrollPickerValue | undefined): void
  (event: 'cancel'): void
  (event: 'wheel', value: ScrollPickerValue | undefined): void
  (event: 'click', value: ScrollPickerValue | undefined): void
}>()

const props = withDefaults(
  defineProps<{
    modelValue: ScrollPickerValue | undefined
    min: number
    max: number
    interval: number
    perGraduationSize?: number
    markedValue?: number // 新增標記值陣列
    dragSensitivity?: number
    touchSensitivity?: number
    wheelSensitivity?: number
    emptyText?: string
  }>(),
  {
    markedValue: undefined,
    perGraduationSize: 3,
    dragSensitivity: 1.7,
    touchSensitivity: 1.7,
    wheelSensitivity: 1,
    emptyText: 'No Options Available',
  },
)

const internalOptions = computed<T[]>(() => {
  const options: T[] = []

  for (let i = props.min; i <= props.max; i += props.interval) {
    options.push({ value: i } as T)
  }

  return options
})
const transitionTimeout = ref<ReturnType<typeof setTimeout> | null>(null)

const internalIndex = ref(
  Math.max(
    internalOptions.value.findIndex(
      (option) => option.value == props.modelValue,
    ),
    0,
  ),
)

let itemOffsets: number[] = []

let scrollOffset = 0
let scrollX = 0
let scrollXMax = 0

let gestureState: GestureState | null = null

const el = shallowRef<HTMLDivElement>()

const rotatorRef = shallowRef<HTMLDivElement>()
const layerLeftRef = shallowRef<HTMLDivElement>()
const layerSelectionRef = shallowRef<HTMLDivElement>()
const layerRightRef = shallowRef<HTMLDivElement>()

let resizeObserver: globalThis.ResizeObserver | null = null

watch(
  () => props.modelValue,
  (value) => {
    const nextIndex = internalOptions.value.findIndex(
      (option) => option.value == value,
    )
    if (internalIndex.value !== nextIndex) {
      internalIndex.value = Math.max(nextIndex, 0)
      scrollTo(findScrollByIndex(nextIndex))
    }
  },
)

watch(internalOptions, (options) => {
  internalIndex.value = Math.max(
    options.findIndex((option) => option.value == props.modelValue),
    0,
  )
  nextTick(() => {
    updateItemOffsets()
    setScroll(findScrollByIndex(internalIndex.value))
  })
})

onMounted(() => {
  if (!el.value) {
    return
  }

  el.value.addEventListener('wheel', handleWheel, EVENT_OPTIONS)
  el.value.addEventListener('touchstart', handleTouchStart, EVENT_OPTIONS)
  el.value.addEventListener('mousedown', handleMouseDown, EVENT_OPTIONS)

  if (typeof window.ResizeObserver !== 'undefined') {
    let raf = null as ReturnType<typeof requestAnimationFrame> | null
    resizeObserver = new window.ResizeObserver(() => {
      if (raf) {
        cancelAnimationFrame(raf)
      }
      raf = requestAnimationFrame(() => {
        updateItemOffsets()
        raf = null
      })
    })
    resizeObserver.observe(el.value)
  } else {
    updateItemOffsets()
  }
})

onBeforeUnmount(() => {
  el.value?.removeEventListener('wheel', handleWheel)
  el.value?.removeEventListener('touchstart', handleTouchStart)
  el.value?.removeEventListener('mousedown', handleMouseDown)

  resizeObserver?.disconnect()
})

function updateItemOffsets() {
  const rotatorEl = rotatorRef.value
  const layerSelectionEl = layerSelectionRef.value
  if (!rotatorEl || !layerSelectionEl) {
    return
  }

  // set scrollOffset
  {
    const { left, right } = layerSelectionEl.getBoundingClientRect()
    const elLeft = el.value?.getBoundingClientRect().left ?? 0
    scrollOffset = (left + right) / 2 - elLeft
  }

  // set itemOffsets
  let firstItemOffset = 0
  itemOffsets = Array.from(rotatorEl.children).map((itemRef, index) => {
    const { left, right } = itemRef.getBoundingClientRect()
    const itemOffset = (left + right) / 2
    if (index === 0) {
      scrollOffset -= itemOffset - left
      firstItemOffset = itemOffset
    }
    return itemOffset - firstItemOffset
  })

  scrollXMax = Math.max(...itemOffsets)
  setScroll(findScrollByIndex(internalIndex.value))
}

function findIndexFromScroll(scroll: number, ignoreDisabled: boolean): number {
  let minDiff = Infinity
  let foundBoundIndex = 0
  for (const [boundIndex, bound] of itemOffsets.entries()) {
    if (!ignoreDisabled && internalOptions.value[boundIndex]?.disabled) {
      continue
    }
    if (Math.abs(bound - scroll) < minDiff) {
      minDiff = Math.abs(bound - scroll)
      foundBoundIndex = boundIndex
    }
  }
  return foundBoundIndex
}

function findScrollByIndex(index: number): number {
  return itemOffsets[Math.min(Math.max(index, 0), itemOffsets.length - 1)]
}

function scrollTo(scroll: number) {
  setScroll(scroll)
  if (transitionTimeout.value) {
    clearTimeout(transitionTimeout.value)
  }
  transitionTimeout.value = setTimeout(() => {
    transitionTimeout.value = null
  }, 150)
}

let scrollRaf = null as ReturnType<typeof requestAnimationFrame> | null
function setScroll(scroll: number) {
  scrollX = scroll
  if (scrollRaf) {
    cancelAnimationFrame(scrollRaf)
  }
  scrollRaf = requestAnimationFrame(() => {
    rotatorRef.value!.style.left = `${scrollOffset - scroll}px`
    scrollRaf = null
  })
  return scroll
}

function bounceEffect(value: number, min: number, max: number, tension = 0.2) {
  if (value < min) {
    return min + (value - min) * tension
  }
  if (value > max) {
    return max + (value - max) * tension
  }
  return value
}

let wheelTimeout: ReturnType<typeof setTimeout> | null = null
function handleWheel(e: WheelEvent) {
  if (!wheelTimeout && scrollX <= 0 && e.deltaX < 0) {
    return
  }
  if (!wheelTimeout && scrollX >= scrollXMax && e.deltaX > 0) {
    return
  }

  if (itemOffsets.length === 1) {
    return
  }
  e.preventDefault()
  // Use deltaX for horizontal scrolling, fall back to deltaY if no horizontal scroll is happening
  const delta = e.deltaX !== 0 ? e.deltaX : e.deltaY
  const scrollXValue = setScroll(
    bounceEffect(scrollX + delta * props.wheelSensitivity, 0, scrollXMax),
  )

  const nextIndex = findIndexFromScroll(scrollXValue, true)
  const nextOption: ScrollPickerOption | undefined =
    internalOptions.value[nextIndex]
  const nextValue = nextOption?.value
  emit('wheel', nextValue)
  if (nextOption && !nextOption.disabled) {
    internalIndex.value = nextIndex
    emitUpdateModelValue(nextValue)
  }

  if (wheelTimeout) {
    clearTimeout(wheelTimeout)
  }
  wheelTimeout = setTimeout(() => {
    scrollTo(findScrollByIndex(findIndexFromScroll(scrollXValue, false)))
    wheelTimeout = null
  }, 100)
}

function handleTouchStart(e: TouchEvent) {
  if (gestureState) {
    return
  }
  if (e.cancelable) {
    e.preventDefault()
  }

  // Track horizontal position for touch
  gestureState = [scrollX, e.touches[0].clientX, false]
  emit('start')

  document.addEventListener('touchmove', handleTouchMove, EVENT_OPTIONS)
  document.addEventListener('touchend', handleTouchEnd, EVENT_OPTIONS)
  document.addEventListener('touchcancel', handleTouchCancel)
}

function handleMouseDown(e: MouseEvent) {
  if (gestureState) {
    return
  }
  if (e.cancelable) {
    e.preventDefault()
  }

  gestureState = [scrollX, e.clientX, false]
  emit('start')

  document.addEventListener('mousemove', handleMouseMove, EVENT_OPTIONS)
  document.addEventListener('mouseup', handleMouseUp, EVENT_OPTIONS)
  document.addEventListener('mouseout', handleMouseOut)
}

function handleTouchMove(e: TouchEvent) {
  if (!gestureState) {
    return
  }
  if (e.cancelable) {
    e.preventDefault()
  }
  // Calculate horizontal difference
  const diff = e.touches[0].clientX - gestureState[1]
  if (Math.abs(diff) > 1.5) {
    const nextGestureState = gestureState.slice() as GestureState
    nextGestureState[2] = true
    gestureState = nextGestureState
  }
  emitMove(
    setScroll(
      bounceEffect(
        gestureState[0] - diff * props.touchSensitivity,
        0,
        scrollXMax,
      ),
    ),
  )
}

function handleMouseMove(e: MouseEvent) {
  if (!gestureState) {
    return
  }
  if (e.cancelable) {
    e.preventDefault()
  }
  const diff = e.clientX - gestureState[1]
  if (Math.abs(diff) > 1.5) {
    const nextGestureState = gestureState.slice() as GestureState
    nextGestureState[2] = true
    gestureState = nextGestureState
  }
  emitMove(
    setScroll(
      bounceEffect(
        gestureState[0] - diff * props.dragSensitivity,
        0,
        scrollXMax,
      ),
    ),
  )
}

function emitMove(scrollX: number) {
  emit(
    'move',
    internalOptions.value[findIndexFromScroll(scrollX, true)]?.value ??
      undefined,
  )
}

function handleMouseUp(e: MouseEvent) {
  if (!gestureState) {
    return
  }
  if (e.cancelable) {
    e.preventDefault()
  }

  endGesture(gestureState[2], e.clientX, e.clientY)
  gestureState = null

  document.removeEventListener('mousemove', handleMouseMove)
  document.removeEventListener('mouseup', handleMouseUp)
  document.removeEventListener('mouseout', handleMouseOut)
}

function handleTouchEnd(e: TouchEvent) {
  if (!gestureState) {
    return
  }
  if (e.cancelable) {
    e.preventDefault()
  }
  endGesture(
    gestureState[2],
    e.changedTouches[0].clientX,
    e.changedTouches[0].clientY,
  )
  gestureState = null

  document.removeEventListener('touchmove', handleTouchMove)
  document.removeEventListener('touchend', handleTouchEnd)
  document.removeEventListener('touchcancel', handleTouchCancel)
}

function endGesture(isDragging: boolean, x: number, y: number) {
  if (isDragging) {
    const nextIndex = findIndexFromScroll(scrollX, false)
    const nextValue = internalOptions.value[nextIndex]?.value ?? null
    scrollTo(findScrollByIndex(nextIndex))
    internalIndex.value = nextIndex
    emit('end', nextValue)
    emitUpdateModelValue(nextValue)
  } else {
    triggerClick(x, y)
  }
}

function triggerClick(x: number, y: number) {
  if (!rotatorRef.value) {
    return
  }

  // Check if clicked on a specific option
  const rotatorChildren = Array.from(rotatorRef.value.children)
  for (let i = 0; i < rotatorChildren.length; i++) {
    const itemEl = rotatorChildren[i] as HTMLElement
    const rect = itemEl.getBoundingClientRect()

    // Check if click is within this option's boundaries
    if (
      rect.left <= x &&
      x <= rect.right &&
      rect.top <= y &&
      y <= rect.bottom
    ) {
      // Don't select disabled options
      if (internalOptions.value[i]?.disabled) {
        continue
      }

      // If item is valid and different from current, select it
      if (internalIndex.value !== i && internalOptions.value[i]) {
        internalIndex.value = i
        const nextValue = internalOptions.value[i].value
        emit('end', nextValue)
        emit('click', nextValue)
        emitUpdateModelValue(nextValue)
        scrollTo(findScrollByIndex(i))
        return
      }
    }
  }

  // If no specific option was clicked, check if clicked on left/right areas
  if (!layerLeftRef.value || !layerRightRef.value) {
    return
  }

  const leftRect = layerLeftRef.value.getBoundingClientRect()
  const rightRect = layerRightRef.value.getBoundingClientRect()
  let nextIndex = internalIndex.value

  if (
    leftRect.left <= x &&
    x <= leftRect.right &&
    leftRect.top <= y &&
    y <= leftRect.bottom
  ) {
    if (internalIndex.value === 0) {
      return // leftmost
    }
    nextIndex--
    while (
      internalOptions.value[nextIndex] &&
      internalOptions.value[nextIndex].disabled
    ) {
      nextIndex--
    }
  } else if (
    rightRect.left <= x &&
    x <= rightRect.right &&
    rightRect.top <= y &&
    y <= rightRect.bottom
  ) {
    if (internalIndex.value === internalOptions.value.length - 1) {
      return // rightmost
    }
    nextIndex++
    while (
      internalOptions.value[nextIndex] &&
      internalOptions.value[nextIndex].disabled
    ) {
      nextIndex++
    }
  }

  if (internalIndex.value !== nextIndex && internalOptions.value[nextIndex]) {
    internalIndex.value = nextIndex
    const nextValue = internalOptions.value[nextIndex].value
    emit('end', nextValue)
    emit('click', nextValue)
    emitUpdateModelValue(nextValue)
    scrollTo(findScrollByIndex(nextIndex))
  }
}

function emitUpdateModelValue(value: ScrollPickerValue) {
  if (props.modelValue !== value) {
    emit('update:modelValue', value)
  }
}

// <Cancel>
function handleMouseOut(e: MouseEvent) {
  if (e.relatedTarget === null) {
    cancelGesture()
  }
}

function handleTouchCancel() {
  cancelGesture()
}

function cancelGesture() {
  scrollTo(findScrollByIndex(internalIndex.value))
  gestureState = null
  emit('cancel')
}

function makerLeft(markedValue: number, optionValue: number) {
  return 50 + ((markedValue - optionValue) / props.interval) * 100
}
// </Cancel>
</script>
<template>
  <div ref="el" class="vue-horizontal-scroll-picker">
    <div
      ref="rotatorRef"
      role="listbox"
      :class="[
        'vue-horizontal-scroll-picker-rotator',
        {
          'vue-horizontal-scroll-picker-rotator-transition': transitionTimeout,
        },
      ]"
    >
      <template
        v-for="(option, optionIndex) in internalOptions"
        :key="optionIndex"
      >
        <div
          role="option"
          class="vue-horizontal-scroll-picker-item"
          :aria-disabled="option.disabled"
          :aria-selected="internalIndex === optionIndex"
          :data-value="option.value ?? ''"
        >
          <div
            v-if="optionIndex % perGraduationSize === 0"
            class="item-label"
            style="position: absolute; bottom: 0"
          >
            <slot name="label" :option="option">{{ option.value }}</slot>
          </div>
          <div class="item-label">●</div>
          <div
            v-if="optionIndex === 0"
            class="vue-horizontal-scroll-picker-marker"
            :style="{
              left: `calc(${makerLeft(markedValue, option.value)}% - 1px)`,
            }"
          >
            <div class="vue-horizontal-scroll-picker-marker-value">
              <slot name="marked" :value="markedValue">{{ markedValue }}</slot>
            </div>
          </div>
        </div>
      </template>
    </div>
    <div class="vue-horizontal-scroll-picker-layer">
      <div
        ref="layerLeftRef"
        class="vue-horizontal-scroll-picker-layer-left"
      ></div>
      <div
        ref="layerSelectionRef"
        class="vue-horizontal-scroll-picker-layer-selection"
      >
        <div class="vue-horizontal-scroll-picker-current-value">
          <slot name="current" :value="internalOptions[internalIndex]?.value">{{
            internalOptions[internalIndex]?.value
          }}</slot>
        </div>
      </div>
      <div
        ref="layerRightRef"
        class="vue-horizontal-scroll-picker-layer-right"
      ></div>
    </div>
  </div>
</template>
