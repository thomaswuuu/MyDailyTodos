<template>
  <div class="tooltip-container" @mouseenter="showTooltip" @mouseleave="hideTooltip">
    <slot></slot>
    <div v-if="isTooltipVisible" class="tooltip">
      {{ text }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';

defineProps<{
  text: string;
}>();

const isTooltipVisible = ref(false);

const showTooltip = () => {
  isTooltipVisible.value = true;
};

const hideTooltip = () => {
  isTooltipVisible.value = false;
};
</script>

<style scoped>
.tooltip-container {
  position: relative;
  display: block;
}

.tooltip {
  position: absolute;
  bottom: 100%;
  left: 50%;
  transform: translateX(-50%);
  margin-bottom: 5px;
  padding: 5px 10px;
  background-color: #333;
  color: #fff;
  border-radius: 4px;
  font-size: 12px;
  white-space: nowrap;
  z-index: 10;
}

@media (max-width: 650px) {
  .tooltip {
    width: 100%;
    left: 0;
    transform: none;
    text-align: center;
    white-space: normal;
  }
}
</style>
