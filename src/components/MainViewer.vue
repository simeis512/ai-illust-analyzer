<template>
  <div class="relative w-full h-full overflow-hidden bg-gray-800 flex justify-center items-center" ref="panzoomContainer">
    <!-- Canvas for image display (affected by Panzoom) -->
    <canvas ref="imageCanvas" class="max-w-full max-h-full object-contain"></canvas>
    <!-- Canvas for disclaimer overlay (unaffected by Panzoom) -->
    <canvas ref="disclaimerCanvas" class="absolute top-0 left-0 w-full h-full pointer-events-none"></canvas>
    <div v-if="!imageSrc && !isLoading" class="text-gray-400 absolute">
      「画像選択」ボタンから画像ファイルを読み込んでください。
    </div>
    <!-- Progress Indicator -->
    <div v-if="isLoading || isProcessing" class="absolute inset-0 bg-gray-900 bg-opacity-75 flex flex-col justify-center items-center z-50">
      <div class="animate-spin rounded-full h-16 w-16 border-b-2 border-white"></div>
      <p class="text-white mt-4">{{ isLoading ? '画像を読み込んでいます...' : '画像を処理中です...' }}</p>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue';
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

const panzoomContainer = ref(null);
const imageCanvas = ref(null);
const disclaimerCanvas = ref(null);
let originalImage = null;
let panzoom = null;

const isLoading = ref(false);
const isProcessing = ref(false);

// --- Disclaimer Drawing ---
function drawDisclaimer() {
  const canvas = disclaimerCanvas.value;
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  const container = panzoomContainer.value;
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
  if (!props.imageSrc || !imageCanvas.value) return;

  isLoading.value = true;
  isProcessing.value = false; // Reset processing flag on new image

  const ctx = imageCanvas.value.getContext('2d');
  const img = new Image();
  img.crossOrigin = 'anonymous';

  img.onload = () => {
    imageCanvas.value.width = img.width;
    imageCanvas.value.height = img.height;
    ctx.drawImage(img, 0, 0);

    // Clean up previous image mat to prevent memory leaks
    if (originalImage && !originalImage.isDeleted()) {
        originalImage.delete();
    }
    originalImage = cv.imread(imageCanvas.value);
    
    isLoading.value = false;
    
    applyProcessing();
    panzoom?.reset();
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
  cv.imshow(imageCanvas.value, originalImage);
}

function applyLevelCorrection() {
  if (!originalImage || originalImage.isDeleted()) return;
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
  cv.imshow(imageCanvas.value, dst);
  
  lut.delete();
  dst.delete();
}

function applyEdgeDetection() {
  if (!originalImage || originalImage.isDeleted()) return;
  let edgeDetectedImage;

  if (props.edgeColorMode === 'mono') {
    edgeDetectedImage = applyMonoEdgeDetection();
  } else {
    edgeDetectedImage = applyColorEdgeDetection();
  }

  if (props.mosaic) {
    const mosaicImage = applyMosaic(edgeDetectedImage, props.mosaicSize);
    cv.imshow(imageCanvas.value, mosaicImage);
    mosaicImage.delete();
  } else {
    cv.imshow(imageCanvas.value, edgeDetectedImage);
  }

  edgeDetectedImage.delete();
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


// --- Lifecycle Hooks ---
onMounted(() => {
  if (panzoomContainer.value && imageCanvas.value) {
    panzoom = Panzoom(imageCanvas.value, {
      maxScale: 100,
      minScale: 1,
      zoomSpeed: 1,
      pinchSpeed: 3,
    });
    panzoomContainer.value.addEventListener('wheel', panzoom.zoomWithWheel);
  }

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