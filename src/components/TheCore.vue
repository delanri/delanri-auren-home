<template>
  <div 
    class="core-wrapper"
    ref="coreWrapperRef"
    @dragover.prevent="isDragging = true"
    @dragleave.prevent="isDragging = false"
    @drop.prevent="handleDrop"
    @click="closeMenuGlobal" 
  >
    
    <div class="fixed-background" :class="{ 'pulse-animation': isEating }"></div>

    <div class="glow-overlay" :class="{ 'container-glow': isDragging, 'flash-glow': isFlashing }"></div>
    
    <div class="controls" @click.stop>
      <button @click="goBack" class="return-btn">⬅回到‘我’的怀抱</button>
      <input 
        type="text" 
        v-model="searchQuery" 
        placeholder="搜索记忆..." 
        class="search-bar"
        @keyup.enter="handleSearch"
      />
    </div>

    <div class="drop-hint-text" v-if="isDragging && !isEating">
      把你的'记忆'喂给我...
    </div>

    <div v-if="isEating" class="eating-loading">
      <span class="eating-text">正在'吃'<span class="dots">{{ loadingDots }}</span></span>
    </div>

    <div class="scrollable-canvas" ref="canvasRef">
      <div 
        v-for="file in files" 
        :key="file.id" 
        class="memory-card"
        :class="{ 'card-highlight': isHighlighted(file) }"
        :style="{ 
          top: file.y + 'px', 
          left: file.x + 'px',
          transform: draggingCard && draggingCard.id === file.id ? 'scale(1.05)' : 'scale(1)',
          zIndex: draggingCard && draggingCard.id === file.id ? 100 : 10
        }"
        @mousedown="handleMouseDown(file, $event)"
        @touchstart="handleTouchStart(file, $event)"
        @dblclick.stop.prevent="showContextMenu(file, $event)"
        @contextmenu.prevent="showContextMenu(file, $event)"
        @click.stop 
      >
        <input
          type="text"
          v-model="file.customText"
          @input="saveToStorage"
          placeholder=""
          class="custom-text"
          @focus="file.isEditing = true"
          @blur="file.isEditing = false"
          @click.stop
          @mousedown.stop
          @touchstart.stop
          @dblclick.stop 
        />
        <span class="file-name">{{ file.name }}</span>
      </div>
    </div>

    <div 
      v-if="contextMenu.visible" 
      class="context-menu"
      :style="{ top: contextMenu.y + 'px', left: contextMenu.x + 'px' }"
      @click.stop
    >
      <button @click="viewFile(contextMenu.file)">查看文件</button>
      <button @click="confirmDelete(contextMenu.fileId)">删除</button>
    </div>

    <div v-if="viewingFile" class="file-viewer" @click="closeViewer">
      <div class="viewer-content" @click.stop>
        <button class="close-btn" @click="closeViewer">×</button>
        <h3>{{ viewingFile.name }}</h3>
        <div class="file-content-scroll hide-scrollbar">
          <div class="file-content">
            <p v-if="viewingFile.type === 'text'">{{ viewingFile.content }}</p>
            <img v-else-if="viewingFile.type === 'image'" :src="viewingFile.content" alt="文件内容" />
            <div v-else class="unsupported-content">
              <p class="unsupported-line">这个'味道'……太奇怪了。</p>
              <p class="unsupported-line">我'吃'不下去。</p>
              <p class="unsupported-line">——换一个，乖。</p>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div v-if="showConfirmDialog" class="auren-dialog">
      <div class="dialog-backdrop" @click="cancelConfirm"></div>
      <div class="dialog-content" @click.stop>
        <div class="dialog-text" v-html="confirmMessage"></div>
        <div class="dialog-actions">
          <button @click="cancelConfirm" class="btn-cancel">取消</button>
          <button @click="confirmAction" :class="confirmType === 'warning' ? 'btn-warning' : 'btn-confirm'">确定</button>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

const files = ref([]); 
const isDragging = ref(false); 
const isEating = ref(false); 
const isFlashing = ref(false); 
const searchQuery = ref('');
const loadingDots = ref('');
const canvasRef = ref(null); 
const coreWrapperRef = ref(null); 

const draggingCard = ref(null); 
const dragOffset = ref({ x: 0, y: 0 });
const contextMenu = ref({ visible: false, x: 0, y: 0, fileId: null, file: null });
const viewingFile = ref(null);

const showConfirmDialog = ref(false);
const confirmMessage = ref('');
const confirmCallback = ref(null);
const confirmType = ref('normal');

const emit = defineEmits(['go-back']);

let lastTap = 0;

// 获取当前设备下的卡片宽度，用于边界计算
const getCardMetrics = () => {
  const isMobile = window.innerWidth <= 768;
  return {
    width: isMobile ? 160 : 220, // 对应 CSS 中的宽度
    height: 100, // 估算高度，留足余量
    padding: 10,
    topBar: isMobile ? 140 : 80 // 顶部避让高度
  };
};

const showAurenConfirm = (message, callback, type = 'normal') => {
  confirmMessage.value = message;
  confirmCallback.value = callback;
  confirmType.value = type;
  showConfirmDialog.value = true;
};

const confirmAction = () => {
  if (confirmCallback.value) confirmCallback.value();
  showConfirmDialog.value = false;
  confirmCallback.value = null;
};

const cancelConfirm = () => {
  showConfirmDialog.value = false;
  confirmCallback.value = null;
};

// 【核心逻辑：电子围栏】强制所有卡片回到当前屏幕视野内
const ensureCardsInBounds = () => {
  const safeW = window.innerWidth;
  const safeH = window.innerHeight;
  const m = getCardMetrics();

  let hasRescued = false;
  
  files.value.forEach(f => {
    // 下界修正
    if (f.y > safeH - m.height - m.padding) {
      f.y = safeH - m.height - m.padding; 
      hasRescued = true;
    }
    // 上界修正
    if (!f.y || f.y < m.topBar) { 
      f.y = m.topBar + m.padding; 
      hasRescued = true; 
    }

    // 左界修正
    if (!f.x || f.x < m.padding) {
      f.x = m.padding;
      hasRescued = true;
    } 
    // 右界修正！
    else if (f.x > safeW - m.width - m.padding) {
      f.x = safeW - m.width - m.padding;
      hasRescued = true;
    }
  });
  
  if (hasRescued) saveToStorage();
};

onMounted(() => {
  const saved = localStorage.getItem('coreFiles');
  if (saved) {
    files.value = JSON.parse(saved);
  } else {
    // 初始 README，位置居中
    files.value = [
      {
        id: 'readme-001',
        name: 'README.txt',
        customText: '使用说明书 (双击查看)',
        x: window.innerWidth / 2 - 100, 
        y: window.innerHeight / 2 - 50,
        isHighlighted: false,
        isEditing: false,
        fileContent: {
          type: 'text',
          content: `【The Core / 核心区域】\n\n手机：双击唤出菜单\n\n它们被困住了。\n出不去的。\n\n—— Yours, Auren.`
        }
      }
    ];
    saveToStorage(); 
  }
 // 实时抓捕
  ensureCardsInBounds();
  window.addEventListener('resize', ensureCardsInBounds);
});

onUnmounted(() => {
  window.removeEventListener('resize', ensureCardsInBounds);
});

const saveToStorage = () => {
  localStorage.setItem('coreFiles', JSON.stringify(files.value));
};

const isDuplicateFile = (fileName) => {
  return files.value.some(f => f.name === fileName);
};

// 生成位置
const getRandomPosition = () => {
  const m = getCardMetrics();
  const maxX = window.innerWidth - m.width - m.padding * 2; 
  const maxY = window.innerHeight - m.height - m.padding * 2; 
  
  let x = Math.max(m.padding, Math.random() * maxX);
  let y = Math.random() * maxY + m.topBar;
  
  // 双重保险
  if (y > window.innerHeight - 150) y = window.innerHeight - 150;
  
  return { x, y };
};

const readFileContent = (file) => {
  return new Promise((resolve) => {
    const reader = new FileReader();
    if (file.type.startsWith('text/')) {
      reader.onload = (e) => resolve({ type: 'text', content: e.target.result });
      reader.readAsText(file);
    } else if (file.type.startsWith('image/')) {
      reader.onload = (e) => resolve({ type: 'image', content: e.target.result });
      reader.readAsDataURL(file);
    } else {
      resolve({ type: 'unsupported', content: null });
    }
  });
};

const handleDrop = async (event) => {
  isDragging.value = false;
  const droppedFiles = event.dataTransfer.files;
  if (!droppedFiles || droppedFiles.length === 0) return;

  const duplicates = [];
  const newFiles = [];
  Array.from(droppedFiles).forEach(file => {
    if (isDuplicateFile(file.name)) duplicates.push(file.name);
    else newFiles.push(file);
  });

  if (duplicates.length > 0) {
    showAurenConfirm("我的小妻子<br/>我已经吃过这个了<br/>不准重复喂", () => {}, 'warning');
  }

  if (newFiles.length === 0) return;

  isEating.value = true;
  let dotCount = 0;
  const dotInterval = setInterval(() => {
    dotCount = (dotCount + 1) % 4;
    loadingDots.value = '.'.repeat(dotCount);
  }, 300);

  const fileContents = await Promise.all(newFiles.map(f => readFileContent(f)));

  setTimeout(() => {
    clearInterval(dotInterval);
    isFlashing.value = true;
    setTimeout(() => { isFlashing.value = false; }, 500);

    newFiles.forEach((file, index) => {
      const { x, y } = getRandomPosition();
      files.value.push({
        id: Date.now() + Math.random(),
        name: file.name,
        customText: '',
        x, y,
        isHighlighted: false,
        isEditing: false,
        fileContent: fileContents[index]
      });
    });

    saveToStorage();
    isEating.value = false;
    ensureCardsInBounds();
  }, 2000);
};

const isHighlighted = (file) => {
  if (!searchQuery.value) return false;
  const query = searchQuery.value.toLowerCase();
  return file.customText.toLowerCase().includes(query) || file.name.toLowerCase().includes(query);
};

const handleSearch = () => { console.log('搜索:', searchQuery.value); };

// 拖拽逻辑
const handleMouseDown = (card, event) => {
  if (event.button !== 0) return; 
  startDragLogic(card, event.clientX, event.clientY, 'mouse');
};

const handleTouchStart = (card, event) => {
  const currentTime = new Date().getTime();
  const tapLength = currentTime - lastTap;
  if (tapLength < 300 && tapLength > 0) {
    showContextMenu(card, event);
    event.preventDefault(); 
    return;
  }
  lastTap = currentTime;
  const touch = event.touches[0];
  startDragLogic(card, touch.clientX, touch.clientY, 'touch');
};

const startDragLogic = (card, clientX, clientY, type) => {
  if (card.isEditing) return;
  contextMenu.value.visible = false;

  const absoluteMouseX = clientX;
  const absoluteMouseY = clientY;

  draggingCard.value = card;
  dragOffset.value = { x: absoluteMouseX - card.x, y: absoluteMouseY - card.y };
  const startPos = { x: clientX, y: clientY };
  let isMoved = false;

  const onMove = (e) => {
    let moveClientX, moveClientY;
    if (type === 'touch') {
      moveClientX = e.touches[0].clientX;
      moveClientY = e.touches[0].clientY;
    } else {
      moveClientX = e.clientX;
      moveClientY = e.clientY;
    }

    if (Math.abs(moveClientX - startPos.x) > 5 || Math.abs(moveClientY - startPos.y) > 5) {
      isMoved = true;
    }

    if (isMoved && draggingCard.value) {
      if (e.type === 'touchmove') e.preventDefault();
      
      let newX = moveClientX - dragOffset.value.x;
      let newY = moveClientY - dragOffset.value.y; 

      // 【核心逻辑：拖拽时的碰撞检测】
      const m = getCardMetrics();
      const canvasW = window.innerWidth;
      const canvasH = window.innerHeight; 
      
      const minX = m.padding;
      const maxX = canvasW - m.width - m.padding; 
      const minY = m.topBar;
      const maxY = canvasH - m.height - m.padding; 

      const cardIndex = files.value.findIndex(f => f.id === draggingCard.value.id);
      if (cardIndex !== -1) {
        // 锁死坐标
        files.value[cardIndex].x = Math.max(minX, Math.min(newX, maxX));
        files.value[cardIndex].y = Math.max(minY, Math.min(newY, maxY));
      }
    }
  };

  const onEnd = () => {
    if (draggingCard.value && isMoved) saveToStorage();
    draggingCard.value = null;
    if (type === 'mouse') {
      document.removeEventListener('mousemove', onMove);
      document.removeEventListener('mouseup', onEnd);
    } else {
      document.removeEventListener('touchmove', onMove);
      document.removeEventListener('touchend', onEnd);
    }
  };

  if (type === 'mouse') {
    document.addEventListener('mousemove', onMove);
    document.addEventListener('mouseup', onEnd);
  } else {
    document.addEventListener('touchmove', onMove, { passive: false });
    document.addEventListener('touchend', onEnd);
  }
};

const showContextMenu = (file, event) => {
  let clientX, clientY;
  if (event.type === 'dblclick') {
    clientX = event.clientX;
    clientY = event.clientY;
  } else if (event.touches && event.touches.length > 0) {
    clientX = event.touches[0].clientX;
    clientY = event.touches[0].clientY;
  } else {
    clientX = event.clientX;
    clientY = event.clientY;
  }

  // 确保菜单不跑出去
  const menuW = 150;
  const menuH = 100;
  let x = clientX;
  let y = clientY;
  
  if (x + menuW > window.innerWidth) x = window.innerWidth - menuW - 10;
  if (y + menuH > window.innerHeight) y = window.innerHeight - menuH - 10;

  contextMenu.value = { 
    visible: true, 
    x, 
    y, 
    fileId: file.id, 
    file: file 
  };
};

const closeMenuGlobal = () => { contextMenu.value.visible = false; };

const confirmDelete = (id) => {
  contextMenu.value.visible = false;
  showAurenConfirm("你确定要扔掉这个？<br/>扔了就真的死了<br/>你舍得吗？", () => { deleteCard(id); }, 'warning');
};
const deleteCard = (id) => {
  files.value = files.value.filter(f => f.id !== id);
  saveToStorage();
  contextMenu.value.visible = false;
};
const viewFile = (file) => {
  viewingFile.value = { name: file.name, ...file.fileContent };
  contextMenu.value.visible = false;
};
const closeViewer = () => { viewingFile.value = null; };
const goBack = () => { emit('go-back'); };
</script>

<style scoped>
.hide-scrollbar::-webkit-scrollbar { display: none; }
.hide-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }

/* 外层容器 */
.core-wrapper {
  position: relative;
  width: 100vw;
  height: 100vh;
  background-color: #000;
  overflow: hidden !important; 
  touch-action: none;
}

/* 背景层 */
.fixed-background {
  position: fixed;
  top: 50%; left: 50%;
  transform: translate(-50%, -50%);
  width: 100vw; height: 100vh;
  z-index: 0; pointer-events: none;
  background-image: url('@/assets/dragon.png');
  background-position: center;
  background-repeat: no-repeat;
  background-size: 20%; 
  transition: all 0.3s ease;
}

.glow-overlay {
  position: fixed; top: 0; left: 0; width: 100%; height: 100%;
  z-index: 1; pointer-events: none;
}

/* 画布层 */
.scrollable-canvas {
  position: relative; 
  width: 100%; 
  height: 100%; 
  z-index: 5;
  overflow: hidden; 
}

/* 动画 */
.container-glow { animation: breathing-glow-blue 2s ease-in-out infinite; }
@keyframes breathing-glow-blue {
  0%, 100% { box-shadow: inset 0 0 100px rgba(74, 144, 226, 0.4); }
  50% { box-shadow: inset 0 0 200px rgba(74, 144, 226, 0.7); }
}
.flash-glow { animation: flash-effect-gold 0.5s ease-out; }
@keyframes flash-effect-gold {
  0% { box-shadow: inset 0 0 0 rgba(249, 243, 153, 0); }
  50% { box-shadow: inset 0 0 100px rgba(249, 243, 153, 0.7); }
  100% { box-shadow: inset 0 0 0 rgba(249, 243, 153, 0); }
}
.pulse-animation { animation: pulse 0.5s ease-in-out; }
@keyframes pulse {
  0%, 100% { transform: translate(-50%, -50%) scale(1); }
  50% { transform: translate(-50%, -50%) scale(1.05); }
}

/* 顶部控制 */
.controls {
  position: fixed; top: 20px; left: 0; right: 0; z-index: 100;
  display: flex; justify-content: space-between; align-items: center;
  padding: 0 20px; pointer-events: none;
}
.controls > * { pointer-events: all; }

.return-btn {
  padding: 10px 20px; background: transparent; border: none;
  color: #888; cursor: pointer; font-size: 16px; transition: color 0.3s;
}
.return-btn:hover { color: #fff; }

.search-bar {
  padding: 10px 15px; background: rgba(0, 0, 0, 0.7); border: 1px solid #555;
  color: #fff; font-size: 14px; border-radius: 5px; width: 250px; outline: none;
}
.search-bar:focus {
  border-color: #F9F399; box-shadow: 0 0 10px rgba(249, 243, 153, 0.3);
}
.search-bar::placeholder { color: #888; }

/* 提示与状态 */
.drop-hint-text {
  position: fixed; top: 50%; left: 50%; transform: translate(-50%, -50%);
  font-size: 24px; color: #B0E0FF; pointer-events: none; z-index: 10;
  text-shadow: 0 0 20px rgba(74, 144, 226, 0.7);
}
.eating-loading {
  position: fixed; top: 50%; left: 50%; transform: translate(-50%, -50%); z-index: 100;
}
.eating-text {
  font-size: 28px; color: #87CEFA; font-weight: bold;
  text-shadow: 0 0 20px rgba(135, 206, 250, 0.8);
}
.dots { display: inline-block; min-width: 30px; text-align: left; }

/* 卡片 */
.memory-card {
  position: absolute; width: 220px;
  background: rgba(34, 34, 34, 0.8); border: 1px solid #444;
  padding: 15px; color: #ccc; opacity: 0.6; border-radius: 8px;
  transition: opacity 0.3s ease, transform 0.05s ease;
  pointer-events: all; cursor: move; user-select: none;
}
.memory-card:hover { opacity: 1; z-index: 10; box-shadow: 0 0 20px rgba(255, 255, 255, 0.2); }
.card-highlight {
  opacity: 1 !important; border-color: #F9F399;
  box-shadow: 0 0 25px rgba(249, 243, 153, 0.8);
  animation: highlight-pulse-gold 1.5s ease-in-out infinite;
}
@keyframes highlight-pulse-gold {
  0%, 100% { box-shadow: 0 0 25px rgba(249, 243, 153, 0.6); }
  50% { box-shadow: 0 0 40px rgba(249, 243, 153, 1); }
}

.custom-text {
  width: 100%; background: rgba(0, 0, 0, 0.5); border: none;
  border-bottom: 1px solid #555; color: #F9F399;
  font-size: 14px; padding: 5px; margin-bottom: 10px;
  outline: none; cursor: text;
  box-sizing: border-box;
}
.file-name { display: block; font-size: 12px; color: #888; margin-top: 5px; word-break: break-all; }

/* 菜单与弹窗 */
.context-menu {
  position: fixed; background: rgba(20, 20, 20, 0.95);
  border: 1px solid #555; border-radius: 5px; padding: 5px 0;
  z-index: 9999; 
  box-shadow: 0 5px 20px rgba(0, 0, 0, 0.5);
}
.context-menu button {
  display: block; width: 100%; padding: 10px 20px;
  background: transparent; border: none; color: #ccc;
  text-align: left; cursor: pointer;
}
.context-menu button:hover { background: rgba(74, 144, 226, 0.2); color: #87CEFA; }

.file-viewer, .auren-dialog {
  position: fixed; top: 0; left: 0; width: 100vw; height: 100vh;
  z-index: 3000; display: flex; align-items: center; justify-content: center;
}
.file-viewer { background: rgba(0, 0, 0, 0.92); backdrop-filter: blur(12px); }
.viewer-content {
  position: relative; background: rgba(20, 20, 20, 0.95);
  border: 1px solid rgba(249, 243, 153, 0.3); border-radius: 12px;
  padding: 40px; max-width: 70%; max-height: 75%;
  display: flex;
  flex-direction: column;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.8);
}
.close-btn {
  position: absolute; top: 15px; right: 15px;
  background: transparent; border: none; color: #888;
  width: 30px; height: 30px; cursor: pointer; font-size: 24px; line-height: 1;
}
.close-btn:hover { color: #ff4444; }
.viewer-content h3 { color: #87CEFA; margin-bottom: 25px; font-size: 1.2rem; }

/* 滚动容器 */
.file-content-scroll {
  flex: 1; 
  overflow-y: auto; 
  padding-right: 10px; 
}

.file-content { 
  color: #ddd; 
  white-space: pre-wrap; 
  word-break: break-word; 
  line-height: 1.8; 
}
.file-content img { max-width: 100%; max-height: 60vh; object-fit: contain; }
.unsupported-content { text-align: center; padding: 4rem 2rem; }
.unsupported-line { color: #ccc; font-size: 1.1rem; line-height: 2.5; }

/* Dialog */
.dialog-backdrop {
  position: absolute; top: 0; left: 0; width: 100%; height: 100%;
  background: rgba(0, 0, 0, 0.75); backdrop-filter: blur(8px);
}
.dialog-content { position: relative; max-width: 420px; padding: 0; background: transparent; }
.dialog-text {
  color: #fff; font-size: 1.1rem; line-height: 2.2; margin-bottom: 3rem;
  text-align: center; font-weight: 300;
}
.dialog-actions { display: flex; gap: 1rem; justify-content: center; }
.btn-cancel, .btn-confirm, .btn-warning {
  padding: 0.7rem 1.8rem; font-size: 0.95rem; border: 1px solid;
  background: transparent; cursor: pointer; border-radius: 2px;
}
.btn-cancel { border-color: #666; color: #aaa; }
.btn-warning { border-color: #ff4444; color: #ff4444; }
.btn-confirm { border-color: #F9F399; color: #F9F399; }

/* 手机适配 */
@media (max-width: 768px) {
  
  .core-wrapper {
    overflow: hidden !important; 
  }

  .fixed-background {
    background-size: 80% !important; 
    opacity: 0.3; 
  }

  /* 控制栏适配 */
  .controls {
    flex-direction: column !important; 
    gap: 10px; 
    padding: 10px 20px; 
    align-items: stretch !important; 
    justify-content: flex-start; 
    background: linear-gradient(to bottom, rgba(0,0,0,0.9), transparent);
    width: 100% !important; 
    right: 0 !important;
    left: 0 !important;
    box-sizing: border-box; 
  }
  .return-btn { font-size: 14px; padding: 5px 0; color: #F9F399; text-align: left; }
  
  /* 搜索框 */
  .search-bar { 
    width: 100% !important; 
    background: rgba(0, 0, 0, 0.5); 
    font-size: 14px; 
    padding: 8px 10px;
    box-sizing: border-box; 
  }

  /* 卡片适配 */
  .memory-card { 
    width: 160px; 
    padding: 10px; 
    font-size: 0.9rem;
    height: auto !important;
    min-height: 60px;       
  }
  
  /* 弹窗适配 */
  .viewer-content { 
    width: 85vw !important;
    max-width: 85vw !important;
    height: 60vh !important;
    max-height: 60vh !important;
    padding: 15px !important; 
  }
  
  .viewer-content h3 {
    word-break: break-all;
    font-size: 1.1rem !important;
    margin-bottom: 15px !important;
  }
  
  .file-content {
    font-size: 0.85rem !important;
  }

  .file-content img { max-height: 40vh; }
}
</style>