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
      <div v-else-if="mode === 'edge'" class="flex items-center space-x-2">
        <button 
          @click="emit('update:edgeColorMode', 'color')" 
          :class="['px-4 py-2 text-sm font-medium rounded-md', edgeColorMode === 'color' ? 'bg-blue-500 text-white' : 'text-gray-700 bg-white hover:bg-gray-50']">
          カラー
        </button>
        <button 
          @click="emit('update:edgeColorMode', 'mono')" 
          :class="['px-4 py-2 text-sm font-medium rounded-md', edgeColorMode === 'mono' ? 'bg-blue-500 text-white' : 'text-gray-700 bg-white hover:bg-gray-50']">
          モノクロ
        </button>
      </div>
    </div>
    <div class="text-xs text-gray-500 text-right px-4 pb-1">
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
});
const emit = defineEmits(['update:levelCenter', 'update:levelRange', 'update:edgeColorMode']);

// Function to calculate the position of the output bubble
const getOutputPosition = (value, min, max) => {
  const percent = (value - min) / (max - min);
  // Calculate the position and apply an offset to center the bubble over the thumb
  // The offset is based on the thumb's width (16px)
  const thumbWidth = 28; // Corresponds to w-5 class
  const offset = (0.5 - percent) * thumbWidth;
  return {
    left: `calc(${percent * 100}% + ${offset}px)`,
  };
};

const levelCenterOutputPosition = computed(() => getOutputPosition(props.levelCenter, 0, 255));
const levelRangeOutputPosition = computed(() => getOutputPosition(props.levelRange, 2, 8));
</script>