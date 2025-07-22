<template>
  <div class="fixed inset-0 z-50 flex items-center justify-center bg-black/50">
    <div class="bg-white rounded-lg shadow-xl p-6 max-w-sm mx-auto">
      <h3 class="text-lg font-bold mb-4">ご注意</h3>
      <div class="text-sm text-gray-700 mb-6 space-y-2">
        <p>この分析結果はあくまで参考情報であり、AI生成か否かを断定するものではありません。</p>
        <p>JPEG圧縮や作風により、AI生成特有のノイズと類似して見える場合があります。</p>
        <p>本ツールの使用によって生じたいかなるトラブルや損害などについても、開発者は責任を負いません。</p>
      </div>
      <button
        @click="$emit('close')"
        class="w-full bg-blue-500 text-white py-2 px-4 rounded-md hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-opacity-50 cursor-pointer"
      >
        了解
      </button>
    </div>
  </div>
</template>

<script setup>
defineEmits(['close']);
</script>
