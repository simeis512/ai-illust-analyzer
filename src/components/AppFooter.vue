<template>
  <footer class="flex-shrink-0 bg-gray-100 border-t border-gray-200">
    <div class="flex flex-col items-center justify-center p-4 pb-6 h-20">
      <!-- Level Correction Controls -->
      <div v-if="mode === 'contrast'" class="w-full max-w-md space-y-2">
        <!-- Level Center Slider -->
        <div class="slider-container flex items-center space-x-4">
          <label for="levelCenter" class="font-medium text-gray-700 text-sm w-12 text-right">中心</label>
          <div class="relative flex-grow">
            <output class="slider-output" :style="levelCenterOutputPosition">
              {{ levelCenter }}
            </output>
            <input 
              type="range" 
              id="levelCenter"
              :value="levelCenter" 
              @input="emit('update:levelCenter', parseInt($event.target.value))" 
              min="0" 
              max="255" 
              step="1"
              class="w-full custom-slider"
            />
          </div>
        </div>
        <!-- Level Range Slider -->
        <div class="slider-container flex items-center space-x-4">
          <label for="levelRange" class="font-medium text-gray-700 text-sm w-12 text-right">範囲</label>
          <div class="relative flex-grow">
             <output class="slider-output" :style="levelRangeOutputPosition">
              {{ levelRange }}
            </output>
            <input 
              type="range" 
              id="levelRange"
              :value="levelRange" 
              @input="emit('update:levelRange', parseInt($event.target.value))" 
              min="2" 
              max="8" 
              step="1"
              class="w-full custom-slider level-range-slider"
            />
          </div>
        </div>
      </div>

      <!-- Edge Detection Controls -->
      <div v-else-if="mode === 'edge'" class="edge-controls-container">
        <!-- Left side of the center divide -->
        <div class="flex justify-end items-center space-x-2">
          <div :class="['channel-toggles', { 'disabled': edgeColorMode === 'mono' }]">
            <label v-for="(enabled, channel) in edgeColorChannels" :key="channel" 
                  :class="['channel-toggle', `channel-${channel}`, { 'toggled-on': enabled }]">
              <input type="checkbox" 
                    :checked="enabled" 
                    @change="toggleChannel(channel)"
                    :disabled="edgeColorMode === 'mono'"
                    class="hidden">
              <span class="channel-letter">{{ channel.toUpperCase() }}</span>
            </label>
          </div>
          <div class="overlap-toggle-wrapper">
            <div v-if="overlapTarget" :class="['overlap-toggle-container', { 'disabled': edgeColorMode === 'mono' }]">
              <label :class="['overlap-toggle', `overlap-${overlapTarget.key.toLowerCase()}`, { 'toggled-on': !hideOverlap }]">
                  <input type="checkbox" 
                        :checked="!hideOverlap" 
                        @change="emit('update:hideOverlap', !$event.target.checked)" 
                        :disabled="edgeColorMode === 'mono'"
                        class="hidden">
                  <span class="overlap-letter">{{ overlapTarget.key }}</span>
              </label>
            </div>
          </div>
          <button 
            @click="emit('update:edgeColorMode', 'color')" 
            :class="['mode-button', { 'active': edgeColorMode === 'color' }]">
            カラー
          </button>
        </div>
        <!-- Right side of the center divide -->
        <div class="flex justify-start items-center">
          <button 
            @click="emit('update:edgeColorMode', 'mono')" 
            :class="['mode-button', { 'active': edgeColorMode === 'mono' }]">
            モノクロ
          </button>
        </div>
      </div>
    </div>
    <div class="text-xs text-gray-500 text-right px-1 pb-1">
      すべての処理はブラウザ内で完結し、画像が外部に送信されることはありません。
    </div>
  </footer>
</template>

<script setup>
import { computed } from 'vue';

const props = defineProps({ 
  mode: String,
  levelCenter: Number,
  levelRange: Number,
  edgeColorMode: String,
  edgeColorChannels: Object,
  hideOverlap: Boolean,
});
const emit = defineEmits(['update:levelCenter', 'update:levelRange', 'update:edgeColorMode', 'update:edgeColorChannels', 'update:hideOverlap']);

const overlapTarget = computed(() => {
  const { r, g, b } = props.edgeColorChannels;
  if (r && g && b) return { key: 'W', channels: ['r', 'g', 'b'] };
  if (r && g) return { key: 'Y', channels: ['r', 'g'] };
  if (g && b) return { key: 'C', channels: ['g', 'b'] };
  if (r && b) return { key: 'M', channels: ['r', 'b'] };
  return null;
});

function toggleChannel(channel) {
  if (props.edgeColorMode === 'mono') return;
  const newChannels = { ...props.edgeColorChannels, [channel]: !props.edgeColorChannels[channel] };
  emit('update:edgeColorChannels', newChannels);
}

// Function to calculate the position of the output bubble
const getOutputPosition = (value, min, max) => {
  const percent = (value - min) / (max - min);
  const thumbWidth = 28;
  const offset = (0.5 - percent) * thumbWidth;
  return {
    left: `calc(${percent * 100}% + ${offset}px)`,
  };
};

const levelCenterOutputPosition = computed(() => getOutputPosition(props.levelCenter, 0, 255));
const levelRangeOutputPosition = computed(() => getOutputPosition(props.levelRange, 2, 8));
</script>
