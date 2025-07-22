<template>
  <div class="relative w-full h-full overflow-hidden bg-gray-800 flex justify-center items-center" ref="panzoomContainer">
    <!-- Canvas for image display (affected by Panzoom) -->
    <canvas ref="imageCanvas" class="max-w-full max-h-full object-contain"></canvas>
    <!-- Canvas for disclaimer overlay (unaffected by Panzoom) -->
    <canvas ref="disclaimerCanvas" class="absolute top-0 left-0 w-full h-full pointer-events-none"></canvas>
    <div v-if="!imageSrc" class="text-gray-400 absolute">
      「画像選択」ボタンから画像ファイルを読み込んでください。
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
});

const panzoomContainer = ref(null);
const imageCanvas = ref(null);
const disclaimerCanvas = ref(null);
let originalImage = null;
let panzoom = null;

// --- Disclaimer Drawing ---
function drawDisclaimer() {
  const canvas = disclaimerCanvas.value;
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
  const safetyMargin = 0.95; // Use 95% of canvas size

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
  ctx.globalAlpha = 0.65;
  ctx.font = `bold ${fontSize}px sans-serif`;
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';

  const lineHeight = fontSize * 1.6;
  const totalTextHeight = lines.length * lineHeight;
  let currentY = -totalTextHeight / 2 + lineHeight / 2;

  // --- 1. Draw Shadowed Outline ---
  ctx.shadowColor = 'black';
  ctx.shadowBlur = 8;
  ctx.shadowOffsetX = 0;
  ctx.shadowOffsetY = 0;
  ctx.strokeStyle = 'black';
  ctx.lineWidth = fontSize / 4;
  ctx.lineJoin = 'round';

  for (const line of lines) {
    ctx.strokeText(line, 0, currentY);
    currentY += lineHeight;
  }

  // --- 2. Draw Foreground Text ---
  ctx.shadowColor = 'transparent';
  ctx.fillStyle = 'white';

  currentY = -totalTextHeight / 2 + lineHeight / 2;
  for (const line of lines) {
    ctx.fillText(line, 0, currentY);
    currentY += lineHeight;
  }

  ctx.restore();
}


// --- Image Loading and Initial Drawing ---
function loadImage() {
  if (!props.imageSrc || !imageCanvas.value) return;
  const ctx = imageCanvas.value.getContext('2d');
  const img = new Image();
  img.crossOrigin = 'anonymous';
  img.onload = () => {
    imageCanvas.value.width = img.width;
    imageCanvas.value.height = img.height;
    ctx.drawImage(img, 0, 0);
    originalImage = cv.imread(imageCanvas.value);
    applyProcessing();
    panzoom?.reset();
    
    requestAnimationFrame(drawDisclaimer);
  };
  img.src = props.imageSrc;
}

// --- Image Processing Logic ---
function applyProcessing() {
  if (!originalImage) return;

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
}

function showOriginal() {
  cv.imshow(imageCanvas.value, originalImage);
}

function applyLevelCorrection() {
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
  if (props.edgeColorMode === 'mono') {
    applyMonoEdgeDetection();
  } else {
    applyColorEdgeDetection();
  }
}

function applyMonoEdgeDetection() {
  const srcGray = new cv.Mat();
  cv.cvtColor(originalImage, srcGray, cv.COLOR_RGBA2GRAY, 0);

  const dst = new cv.Mat();
  cv.Laplacian(srcGray, dst, cv.CV_8U, 1, 1, 0, cv.BORDER_DEFAULT);

  const absDst = new cv.Mat();
  cv.convertScaleAbs(dst, absDst);

  const binaryDst = new cv.Mat();
  cv.threshold(absDst, binaryDst, 0, 255, cv.THRESH_BINARY);
  
  cv.imshow(imageCanvas.value, binaryDst);

  srcGray.delete();
  dst.delete();
  absDst.delete();
  binaryDst.delete();
}

function applyColorEdgeDetection() {
  let rgbaPlanes = new cv.MatVector();
  cv.split(originalImage, rgbaPlanes);

  let mergedPlanes = new cv.MatVector();

  for (let i = 0; i < 3; i++) {
    let singleChannel = rgbaPlanes.get(i);
    let dst = new cv.Mat();
    cv.Laplacian(singleChannel, dst, cv.CV_8U, 1, 1, 0, cv.BORDER_DEFAULT);
    let absDst = new cv.Mat();
    cv.convertScaleAbs(dst, absDst);
    let binaryDst = new cv.Mat();
    cv.threshold(absDst, binaryDst, 0, 255, cv.THRESH_BINARY);
    mergedPlanes.push_back(binaryDst);

    singleChannel.delete();
    dst.delete();
    absDst.delete();
    binaryDst.delete();
  }

  mergedPlanes.push_back(rgbaPlanes.get(3));

  const result = new cv.Mat();
  cv.merge(mergedPlanes, result);
  cv.imshow(imageCanvas.value, result);

  rgbaPlanes.delete();
  mergedPlanes.delete();
  result.delete();
}


// --- Watchers ---
watch(() => props.imageSrc, loadImage);
watch(() => [props.mode, props.levelCenter, props.levelRange, props.edgeColorMode], () => {
    applyProcessing();
    requestAnimationFrame(drawDisclaimer);
});


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
