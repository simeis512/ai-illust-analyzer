<template>
  <div ref="rootContainer" class="relative w-full h-full bg-gray-800">
    <div class="absolute inset-0 flex" :class="{ 'flex-row': isSplitView, 'flex-col': !isSplitView }">
      <!-- Left/Single Viewer -->
      <div ref="originalViewer" class="relative overflow-hidden flex-1 justify-center items-center flex">
        <canvas ref="originalCanvas" class="max-w-full max-h-full object-contain"></canvas>
        <div v-if="isSplitView" class="absolute top-2 right-2 bg-gray-800 bg-opacity-50 text-white px-2 py-1 rounded">
          処理前
        </div>
      </div>

      <!-- Right Viewer (only in split view) -->
      <div v-if="isSplitView" ref="processedViewer" class="relative overflow-hidden flex-1 justify-center items-center flex border-l-2 border-gray-600">
        <canvas ref="processedCanvas" class="max-w-full max-h-full object-contain"></canvas>
        <div class="absolute top-2 left-2 bg-gray-800 bg-opacity-50 text-white px-2 py-1 rounded">
          処理後
        </div>
      </div>
    </div>

    <!-- Disclaimer Canvas -->
    <canvas ref="disclaimerCanvas" class="absolute top-0 left-0 w-full h-full pointer-events-none"></canvas>

    <!-- Initial Text -->
    <div v-if="!imageSrc && !isLoading" class="absolute inset-0 flex justify-center items-center text-gray-400">
      「画像選択」ボタンから画像ファイルを読み込んでください。
    </div>

    <!-- Progress Indicator -->
    <div v-if="isLoading || isProcessing" class="absolute inset-0 bg-gray-900 bg-opacity-75 flex flex-col justify-center items-center z-50">
      <div class="animate-spin rounded-full h-16 w-16 border-b-2 border-white"></div>
      <p class="text-white mt-4">{{ isLoading ? '画像を読み込んでいます...' : '画像を処理中です...' }}</p>
    </div>
    
    <!-- Split View Toggle Button -->
    <button 
      v-if="imageSrc" 
      @click="toggleSplitView" 
      class="absolute bottom-4 left-4 bg-gray-700 hover:bg-gray-600 opacity-60 hover:opacity-100 text-white p-2 rounded-full shadow-lg transition-all duration-200 transform hover:scale-105 z-100 cursor-pointer"
      :class="{ 'bg-gray-500': isSplitView }"
      aria-label="Toggle Split View"
    >
      <svg class="split-view-icon" :class="{ 'active': isSplitView }" width="24" height="24" viewBox="0 0 32 32" xmlns="http://www.w3.org/2000/svg">
        <rect class="left-rect" x="2" y="2" width="28" height="28" rx="2" />
        <rect class="right-rect" x="19" y="2" width="13" height="28" rx="2" />
      </svg>
    </button>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, nextTick } from 'vue';
import Panzoom from '@panzoom/panzoom';

const props = defineProps({
  imageSrc: String,
  mode: String,
  levelCenter: Number,
  levelRange: Number,
  edgeColorMode: String,
  edgeColorChannels: Object,
  hideOverlap: Boolean,
  mosaic: Boolean,
  mosaicSize: Number,
});

const rootContainer = ref(null);
const originalViewer = ref(null);
const processedViewer = ref(null);
const originalCanvas = ref(null);
const processedCanvas = ref(null);
const disclaimerCanvas = ref(null);

let originalImage = null;
let panzoomOriginal = null;
let panzoomProcessed = null;
let isSyncing = false; // Flag to prevent event loops

const isLoading = ref(false);
const isProcessing = ref(false);
const isSplitView = ref(false);

function toggleSplitView() {
  isSplitView.value = !isSplitView.value;
  nextTick(() => {
    resetAllPanzooms();
    applyProcessing(); // Re-apply processing to ensure the correct canvas is updated
  });
}


// --- Disclaimer Drawing ---
function drawDisclaimer() {
  const canvas = disclaimerCanvas.value;
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  const container = rootContainer.value; // Use the root container for sizing
  if (!container) return;

  // Ensure the disclaimer canvas has the same dimensions as the root container.
  if (container.clientWidth !== canvas.width || container.clientHeight !== canvas.height) {
      canvas.width = container.clientWidth;
      canvas.height = container.clientHeight;
  }
  
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  const lines = [
    '【ご注意】',
    'この分析結果はあくまで参考情報であり、AI生成か否かを断定するものではありません。',
    'JPEG圧縮や作風により、AI生成特有のノイズと類似して見える場合があります。',
    '本ツールの使用によって生じたいかなるトラブルや損害などについても、開発者は責任を負いません。'
  ];

  // --- Find Optimal Font Size ---
  let fontSize = Math.floor(canvas.height / 4); // Start with a large font size
  let textWidth, textHeight, boxWidth, boxHeight;
  const angle = Math.atan2(canvas.height, canvas.width);
  const safetyMargin = 0.98; // Use 98% of canvas size

  while (fontSize > 10) { // Minimum font size of 10
    const lineHeight = fontSize * 1.6;
    ctx.font = `bold ${fontSize}px sans-serif`;
    
    // Find the longest line width
    textWidth = 0;
    for (const line of lines) {
        textWidth = Math.max(textWidth, ctx.measureText(line).width);
    }
    textHeight = lines.length * lineHeight;

    // Calculate the bounding box of the rotated text
    boxWidth = Math.abs(textWidth * Math.cos(angle)) + Math.abs(textHeight * Math.sin(angle));
    boxHeight = Math.abs(textWidth * Math.sin(angle)) + Math.abs(textHeight * Math.cos(angle));

    if (boxWidth < canvas.width * safetyMargin && boxHeight < canvas.height * safetyMargin) {
      break; // Found a good font size
    }
    fontSize -= 2; // Decrease font size and try again
  }

  // --- Drawing Logic ---
  ctx.save();
  
  // Move origin to canvas center and rotate
  ctx.translate(canvas.width / 2, canvas.height / 2);
  ctx.rotate(-angle);

  // --- Common Text Style ---
  ctx.globalAlpha = 0.75;
  ctx.font = `bold ${fontSize}px sans-serif`;
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';

  const lineHeight = fontSize * 1.6;
  const titleLine = lines[0];
  const mainLines = lines.slice(1);
  const mainTextHeight = mainLines.length * lineHeight;
  
  // Center the main block of text, then place title above it
  const mainBlockStartY = -mainTextHeight / 2 + lineHeight / 2;
  const titleY = mainBlockStartY - lineHeight;

  // --- 1. Draw Shadowed Outline ---
  ctx.shadowColor = 'black';
  ctx.shadowBlur = 8;
  ctx.shadowOffsetX = 0;
  ctx.shadowOffsetY = 0;
  ctx.strokeStyle = 'black';
  ctx.lineWidth = fontSize / 4;
  ctx.lineJoin = 'round';

  // Draw title
  ctx.strokeText(titleLine, 0, titleY);
  // Draw main lines
  let currentY = mainBlockStartY;
  for (const line of mainLines) {
    ctx.strokeText(line, 0, currentY);
    currentY += lineHeight;
  }

  // --- 2. Draw Foreground Text ---
  ctx.shadowColor = 'transparent';
  ctx.fillStyle = 'white';

  // Draw title
  ctx.fillText(titleLine, 0, titleY);
  // Draw main lines
  currentY = mainBlockStartY;
  for (const line of mainLines) {
    ctx.fillText(line, 0, currentY);
    currentY += lineHeight;
  }

  ctx.restore();
}


// --- Image Loading and Initial Drawing ---
function loadImage() {
  if (!props.imageSrc || !originalCanvas.value) return;

  isLoading.value = true;
  isProcessing.value = false;

  const canvases = [originalCanvas.value, processedCanvas.value].filter(Boolean);
  const img = new Image();
  img.crossOrigin = 'anonymous';

  img.onload = () => {
    for (const canvas of canvases) {
        canvas.width = img.width;
        canvas.height = img.height;
        const ctx = canvas.getContext('2d');
        ctx.drawImage(img, 0, 0);
    }

    if (originalImage && !originalImage.isDeleted()) {
        originalImage.delete();
    }
    originalImage = cv.imread(originalCanvas.value);
    
    isLoading.value = false;
    
    applyProcessing();
    resetAllPanzooms();
  };

  img.onerror = () => {
    isLoading.value = false;
    console.error("Image loading failed.");
    // Optionally, display an error message to the user in the UI
  };

  img.src = props.imageSrc;
}

// --- Image Processing Logic ---
function applyProcessing() {
  if (!originalImage) return;

  if (props.mode === 'edge') {
    isProcessing.value = true;
  }

  // Use setTimeout to allow the UI to update. Set processing flag only for heavy tasks.
  setTimeout(() => {
    try {
      switch (props.mode) {
        case 'contrast':
          applyLevelCorrection();
          break;
        case 'edge':
          applyEdgeDetection();
          break;
        default:
          showOriginal();
          break;
      }
    } catch (e) {
      console.error("An error occurred during image processing:", e);
      showOriginal(); // Fallback to original image on error
    } finally {
      // Reset processing flag if it was set
      if (isProcessing.value) {
        isProcessing.value = false;
      }
      // Redraw disclaimer after processing is complete
      requestAnimationFrame(drawDisclaimer);
    }
  }, 10); // A small delay to ensure the loading indicator is rendered
}

function showOriginal() {
  if (!originalImage || originalImage.isDeleted()) return;
  const targetCanvas = isSplitView.value ? processedCanvas.value : originalCanvas.value;
  if (targetCanvas) {
    cv.imshow(targetCanvas, originalImage);
  }
  // In split view, original canvas should always show the original
  if (isSplitView.value) {
      cv.imshow(originalCanvas.value, originalImage);
  }
}

function applyLevelCorrection() {
  if (!originalImage || originalImage.isDeleted()) return;
  const targetCanvas = isSplitView.value ? processedCanvas.value : originalCanvas.value;
  if (!targetCanvas) return;

  const lut = new cv.Mat(1, 256, cv.CV_8U);
  
  const invertedLevelCenter = 255 - props.levelCenter;
  const range = props.levelRange;

  const halfRangeFloor = Math.floor((range - 1) / 2);
  const halfRangeCeil = Math.ceil((range - 1) / 2);

  const minLevelRaw = invertedLevelCenter - halfRangeFloor;
  const maxLevelRaw = invertedLevelCenter + halfRangeCeil;

  const minLevel = Math.max(0, Math.min(255, minLevelRaw));
  const maxLevel = Math.max(0, Math.min(255, maxLevelRaw));

  const effectiveRange = maxLevel - minLevel;

  if (effectiveRange <= 0) {
    for (let i = 0; i < 256; i++) {
      lut.data[i] = i < minLevel ? 0 : 255;
    }
  } else {
    for (let i = 0; i < 256; i++) {
      if (i <= minLevel) {
        lut.data[i] = 0;
      } else if (i >= maxLevel) {
        lut.data[i] = 255;
      } else {
        const value = ((i - minLevel) / effectiveRange) * 255;
        lut.data[i] = Math.round(value);
      }
    }
  }

  const dst = new cv.Mat();
  cv.LUT(originalImage, lut, dst);
  cv.imshow(targetCanvas, dst);
  
  lut.delete();
  dst.delete();
  
  if (isSplitView.value) {
      cv.imshow(originalCanvas.value, originalImage);
  }
}

function applyEdgeDetection() {
  if (!originalImage || originalImage.isDeleted()) return;
  const targetCanvas = isSplitView.value ? processedCanvas.value : originalCanvas.value;
  if (!targetCanvas) return;

  let edgeDetectedImage;

  if (props.edgeColorMode === 'mono') {
    edgeDetectedImage = applyMonoEdgeDetection();
  } else {
    edgeDetectedImage = applyColorEdgeDetection();
  }

  if (props.mosaic) {
    const mosaicImage = applyMosaic(edgeDetectedImage, props.mosaicSize);
    cv.imshow(targetCanvas, mosaicImage);
    mosaicImage.delete();
  } else {
    cv.imshow(targetCanvas, edgeDetectedImage);
  }

  edgeDetectedImage.delete();
  
  if (isSplitView.value) {
      cv.imshow(originalCanvas.value, originalImage);
  }
}

function applyMonoEdgeDetection() {
  if (!originalImage || originalImage.isDeleted()) return new cv.Mat();
  const srcGray = new cv.Mat();
  cv.cvtColor(originalImage, srcGray, cv.COLOR_RGBA2GRAY, 0);

  const dst = new cv.Mat();
  cv.Laplacian(srcGray, dst, cv.CV_8U, 1, 1, 0, cv.BORDER_DEFAULT);

  const absDst = new cv.Mat();
  cv.convertScaleAbs(dst, absDst);

  const binaryDst = new cv.Mat();
  cv.threshold(absDst, binaryDst, 0, 255, cv.THRESH_BINARY);
  
  // Convert grayscale back to RGBA for consistent handling
  const rgbaDst = new cv.Mat();
  cv.cvtColor(binaryDst, rgbaDst, cv.COLOR_GRAY2RGBA);

  srcGray.delete();
  dst.delete();
  absDst.delete();
  binaryDst.delete();

  return rgbaDst;
}

function applyColorEdgeDetection() {
  if (!originalImage || originalImage.isDeleted()) return new cv.Mat();
  let rgbaPlanes = new cv.MatVector();
  cv.split(originalImage, rgbaPlanes);

  let processedPlanes = new cv.MatVector();
  const channelOrder = ['r', 'g', 'b'];
  let activeChannels = [];

  for (let i = 0; i < 3; i++) {
    const channelKey = channelOrder[i];
    let singleChannel = rgbaPlanes.get(i);
    let processedChannel;

    if (props.edgeColorChannels[channelKey]) {
      activeChannels.push(i);
      let dst = new cv.Mat();
      cv.Laplacian(singleChannel, dst, cv.CV_8U, 1, 1, 0, cv.BORDER_DEFAULT);
      let absDst = new cv.Mat();
      cv.convertScaleAbs(dst, absDst);
      processedChannel = new cv.Mat();
      cv.threshold(absDst, processedChannel, 0, 255, cv.THRESH_BINARY);
      dst.delete();
      absDst.delete();
    } else {
      processedChannel = cv.Mat.zeros(originalImage.rows, originalImage.cols, cv.CV_8U);
    }
    processedPlanes.push_back(processedChannel);
    singleChannel.delete();
  }

  if (props.hideOverlap && activeChannels.length > 1) {
      let overlap = new cv.Mat();
      let firstChannel = processedPlanes.get(activeChannels[0]).clone();
      cv.bitwise_and(firstChannel, processedPlanes.get(activeChannels[1]), overlap);
      if (activeChannels.length === 3) {
          cv.bitwise_and(overlap, processedPlanes.get(activeChannels[2]), overlap);
      }

      for (const i of activeChannels) {
          let currentPlane = processedPlanes.get(i);
          cv.bitwise_xor(currentPlane, overlap, currentPlane);
      }
      overlap.delete();
      firstChannel.delete();
  }

  // Keep original alpha channel
  processedPlanes.push_back(rgbaPlanes.get(3));

  const result = new cv.Mat();
  cv.merge(processedPlanes, result);

  rgbaPlanes.delete();
  processedPlanes.delete();

  return result;
}

function applyMosaic(src, blockSize) {
  const dst = src.clone();
  const cols = src.cols;
  const rows = src.rows;

  const colorMap = {
    '0,0,0': 'K',
    '255,0,0': 'R',
    '0,255,0': 'G',
    '0,0,255': 'B',
    '255,255,0': 'Y',
    '0,255,255': 'C',
    '255,0,255': 'M',
    '255,255,255': 'W'
  };

  const colorToComponent = {
      'R': 0b100,
      'G': 0b010,
      'B': 0b001,
      'Y': 0b110,
      'C': 0b011,
      'M': 0b101,
      'W': 0b111,
      'K': 0b000,
  };

  for (let y = 0; y < rows; y += blockSize) {
    for (let x = 0; x < cols; x += blockSize) {
      const rect = new cv.Rect(x, y, Math.min(blockSize, cols - x), Math.min(blockSize, rows - y));
      
      const histogram = { K: 0, R: 0, G: 0, B: 0, Y: 0, C: 0, M: 0, W: 0 };
      
      for (let j = 0; j < rect.height; j++) {
        for (let i = 0; i < rect.width; i++) {
          // ブロックの左上座標(x, y)とループ変数(i, j)から絶対座標を計算
          const absoluteX = x + i;
          const absoluteY = y + j;
          
          // 元画像のデータから直接インデックスを計算して値を取得
          const index = (absoluteY * src.cols * 4) + (absoluteX * 4);
          const r = src.data[index];
          const g = src.data[index + 1];
          const b = src.data[index + 2];

          const key = `${r},${g},${b}`;
          if (colorMap[key]) {
            histogram[colorMap[key]]++;
          }
        }
      }
      histogram.K /= 8;

      let maxCount = 0;
      for (const color in histogram) {
        if (histogram[color] > maxCount) {
          maxCount = histogram[color];
        }
      }

      let dominantColors = [];
      for (const color in histogram) {
        if (histogram[color] === maxCount) {
          dominantColors.push(color);
        }
      }
      
      let finalColorComponent = 0b000;
      if (dominantColors.length > 0) {
          finalColorComponent = dominantColors.reduce((acc, color) => {
              return acc | colorToComponent[color];
          }, 0b000);
      }

      let finalColor = [0, 0, 0];
      if ((finalColorComponent & 0b100) !== 0) finalColor[0] = 255;
      if ((finalColorComponent & 0b010) !== 0) finalColor[1] = 255;
      if ((finalColorComponent & 0b001) !== 0) finalColor[2] = 255;

      const destRoi = dst.roi(rect);
      const scalar = new cv.Scalar(finalColor[0], finalColor[1], finalColor[2], 255);
      destRoi.setTo(scalar);

      destRoi.delete();
    }
  }
  return dst;
}


// --- Watchers ---
watch(() => props.imageSrc, loadImage);
watch(() => [props.mode, props.levelCenter, props.levelRange, props.edgeColorMode, props.edgeColorChannels, props.hideOverlap, props.mosaic, props.mosaicSize], () => {
    applyProcessing();
}, { deep: true });
watch(isSplitView, () => {
    nextTick(() => {
        initPanzooms();
    });
});

// --- Panzoom Handling ---

// Define handlers so they can be added and removed correctly.
let handleOriginalEvent, handleProcessedEvent;
// Store element references for reliable listener removal, as Vue refs can become null before cleanup.
let originalCanvasEl = null;
let processedCanvasEl = null;

/**
 * Syncs the transform (pan and zoom) from a source Panzoom instance to a target instance.
 * @param {Panzoom} source The Panzoom instance that was directly manipulated.
 * @param {Panzoom} target The Panzoom instance that needs to be synced.
 */
const syncPanzooms = (source, target) => {
    // If there is no target, a sync is already in progress, or the source is invalid, do nothing.
    if (!target || isSyncing || !source) return;
    isSyncing = true;

    const x = source.getPan().x;
    const y = source.getPan().y;
    const scale = source.getScale();

    target.pan(x, y, { animate: false, force: true });
    target.zoom(scale, { animate: false, force: true, noSetRange: true });

    requestAnimationFrame(() => { isSyncing = false; });
};

/**
 * Initializes the Panzoom instances. It destroys existing instances and re-creates them,
 * attaching the necessary event listeners and storing element references for later cleanup.
 */
function initPanzooms() {
    destroyPanzooms();

    // 1. Create Panzoom instances and store element references.
    if (originalViewer.value && originalCanvas.value) {
        panzoomOriginal = Panzoom(originalCanvas.value, { maxScale: 100, minScale: 0.1 });
        originalCanvasEl = originalCanvas.value; // Store ref for cleanup
    }
    if (isSplitView.value && processedViewer.value && processedCanvas.value) {
        panzoomProcessed = Panzoom(processedCanvas.value, { maxScale: 100, minScale: 0.1 });
        processedCanvasEl = processedCanvas.value; // Store ref for cleanup
    }

    // 2. Create and attach event handlers now that instances are guaranteed to exist.
    if (panzoomOriginal) {
        originalViewer.value.addEventListener('wheel', panzoomOriginal.zoomWithWheel);
        handleOriginalEvent = () => syncPanzooms(panzoomOriginal, panzoomProcessed);
        originalCanvasEl.addEventListener('panzoomchange', handleOriginalEvent);
    }
    if (panzoomProcessed) {
        processedViewer.value.addEventListener('wheel', panzoomProcessed.zoomWithWheel);
        handleProcessedEvent = () => syncPanzooms(panzoomProcessed, panzoomOriginal);
        processedCanvasEl.addEventListener('panzoomchange', handleProcessedEvent);
    }
}

/**
 * Resets the pan and zoom to the initial state for all active Panzoom instances.
 */
function resetAllPanzooms() {
    panzoomOriginal?.reset();
    panzoomProcessed?.reset();
    if (isSplitView.value && !panzoomProcessed) {
        nextTick(initPanzooms);
    }
}

/**
 * Properly removes event listeners from stored element references and destroys
 * Panzoom instances to prevent memory leaks.
 */
function destroyPanzooms() {
    if (panzoomOriginal) {
        // panzoom.destroy() removes its own listeners (e.g., wheel on the parent).
        // We only need to manually remove the listeners we added ourselves.
        if (originalCanvasEl && handleOriginalEvent) {
            originalCanvasEl.removeEventListener('panzoomchange', handleOriginalEvent);
        }
        panzoomOriginal.destroy();
        panzoomOriginal = null;
        originalCanvasEl = null;
    }
    if (panzoomProcessed) {
        if (processedCanvasEl && handleProcessedEvent) {
            processedCanvasEl.removeEventListener('panzoomchange', handleProcessedEvent);
        }
        panzoomProcessed.destroy();
        panzoomProcessed = null;
        processedCanvasEl = null;
    }
}


// --- Lifecycle Hooks ---
onMounted(() => {
  initPanzooms();

  const checkCv = setInterval(() => {
    if (typeof cv !== 'undefined' && cv.Mat) {
      clearInterval(checkCv);
      console.log('OpenCV.js is ready.');
      if (props.imageSrc) {
        loadImage();
      }
    }
  }, 100);
  
  // Redraw on window resize
  window.addEventListener('resize', drawDisclaimer);
});
</script>