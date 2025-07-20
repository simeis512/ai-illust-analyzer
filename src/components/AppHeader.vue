<template>
  <header class="flex-shrink-0 bg-gray-100 border-b border-gray-200">
    <div class="flex items-center justify-between p-2">
      <div class="flex items-center space-x-1">
        <div v-if="imageLoaded">
          <button
            @click="emit('set-mode', 'original')"
            :class="['px-3 py-2 text-sm font-medium rounded-md', currentMode === 'original' ? 'bg-blue-500 text-white' : 'text-gray-700 bg-white hover:bg-gray-50', !imageLoaded ? 'opacity-50 cursor-not-allowed' : 'cursor-pointer']"
            :disabled="!imageLoaded">
            オリジナル
          </button>
          <button
            @click="emit('set-mode', 'contrast')"
            :class="['px-3 py-2 text-sm font-medium rounded-md', currentMode === 'contrast' ? 'bg-blue-500 text-white' : 'text-gray-700 bg-white hover:bg-gray-50', !imageLoaded ? 'opacity-50 cursor-not-allowed' : 'cursor-pointer']"
            :disabled="!imageLoaded">
            レベル補正
          </button>
          <button
            @click="emit('set-mode', 'edge')"
            :class="['px-3 py-2 text-sm font-medium rounded-md', currentMode === 'edge' ? 'bg-blue-500 text-white' : 'text-gray-700 bg-white hover:bg-gray-50', !imageLoaded ? 'opacity-50 cursor-not-allowed' : 'cursor-pointer']"
            :disabled="!imageLoaded">
            エッジ抽出
          </button>
        </div>
      </div>
      <div class="actions">
        <input type="file" ref="fileInput" @change="onFileSelect" class="hidden" accept="image/*" />
        <button @click="triggerFileInput" class="px-4 py-2 text-sm font-medium text-white bg-green-500 rounded-md hover:bg-green-600 cursor-pointer">
          画像選択
        </button>
      </div>
    </div>
  </header>
</template>

<script setup>
import { ref } from 'vue';

const props = defineProps({
  currentMode: String,
  imageLoaded: Boolean
});
const emit = defineEmits(['set-mode', 'image-selected']);

const fileInput = ref(null);

function triggerFileInput() {
  fileInput.value.click();
}

function onFileSelect(event) {
  const file = event.target.files[0];
  if (file) {
    const reader = new FileReader();
    reader.onload = (e) => {
      emit('image-selected', e.target.result);
    };
    reader.readAsDataURL(file);
  }
}
</script>
