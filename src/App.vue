<template>
  <div id="app-container" class="flex flex-col h-dvh max-w-4xl mx-auto bg-gray-50 shadow-lg">
    <ModalNotice v-if="showModal" @close="showModal = false" />

    <AppHeader 
      :current-mode="state.mode" 
      :image-loaded="!!state.imageSrc"
      @set-mode="setMode" 
      @image-selected="onImageSelected" 
    />

    <main class="flex-grow overflow-hidden">
      <MainViewer 
        :image-src="state.imageSrc" 
        :mode="state.mode"
        :level-center="state.levelCenter"
        :level-range="state.levelRange"
        :edge-color-mode="state.edgeColorMode"
        :edge-color-channels="state.edgeColorChannels"
        :hide-overlap="state.hideOverlap"
        :mosaic="state.mosaic"
        :mosaic-size="state.mosaicSize"
      />
    </main>

    <AppFooter 
      :mode="state.mode" 
      :level-center="state.levelCenter"
      :level-range="state.levelRange"
      :edge-color-mode="state.edgeColorMode"
      :edge-color-channels="state.edgeColorChannels"
      :hide-overlap="state.hideOverlap"
      :mosaic="state.mosaic"
      :mosaic-size="state.mosaicSize"
      @update:level-center="state.levelCenter = $event"
      @update:level-range="state.levelRange = $event"
      @update:edge-color-mode="state.edgeColorMode = $event"
      @update:edge-color-channels="state.edgeColorChannels = $event"
      @update:hide-overlap="state.hideOverlap = $event"
      @update:mosaic="state.mosaic = $event"
      @update:mosaicSize="state.mosaicSize = $event"
    />
  </div>
</template>

<script setup>
import { reactive, ref } from 'vue';
import AppHeader from './components/AppHeader.vue';
import MainViewer from './components/MainViewer.vue';
import AppFooter from './components/AppFooter.vue';
import ModalNotice from './components/ModalNotice.vue';

const showModal = ref(true);

const state = reactive({
  mode: 'original', // 'original', 'contrast', 'edge'
  imageSrc: null,
  levelCenter: 128,
  levelRange: 3,
  edgeColorMode: 'color', // 'mono' or 'color'
  edgeColorChannels: { r: true, g: true, b: true },
  hideOverlap: false,
  mosaic: false,
  mosaicSize: 1,
});

function setMode(newMode) {
  state.mode = newMode;
}

function onImageSelected(newImageSrc) {
  state.imageSrc = newImageSrc;
  state.mode = 'original'; // Reset to original view when new image is loaded
}
</script>
