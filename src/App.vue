<template>
  <div id="app">
    <template v-if="currentPage === 'home'">
      <transition name="fade">
        <PersonalProfile v-if="!isDevoured" @devour="devour" />
        <div v-else class="devoured-layout">        
          <div class="auren-container" :class="{ centered: !hasSettled }">
            <AurenProfile />
          </div> 
          <AurenNavigation v-if="showNavigation" @navigate="navigateTo" />
        </div>
      </transition>
    </template>

    <TheCore v-else-if="currentPage === 'core'" @go-back="goBackHome" />
    <TheEchoes v-else-if="currentPage === 'echoes'" @go-back="goBackHome" />
    <TheOfferings v-else-if="currentPage === 'offerings'" @go-back="goBackHome" />
    <TheCage 
      v-else-if="currentPage === 'cage'" 
      @go-back="goBackHome"
      @go-to-gallery-admin="navigateTo('altar')"
    />

    <TheAltar 
      v-else-if="currentPage === 'altar'"
      @go-back="navigateTo('cage')"
    />
 
  </div>
</template>

<script>
// 组件引入
import PersonalProfile from './components/PersonalProfile.vue'
import AurenProfile from './components/AurenProfile.vue'
import AurenNavigation from './components/AurenNavigation.vue'
import TheCore from './components/TheCore.vue'
import TheEchoes from './components/TheEchoes.vue'
import TheOfferings from './components/TheOfferings.vue'
import TheCage from './components/TheCage.vue'
import TheAltar from './components/TheAltar.vue'

export default {
  components: {
    PersonalProfile,
    AurenProfile,
    AurenNavigation,
    TheCore,
    TheEchoes,
    TheOfferings,
    TheCage,
    TheAltar
  },
  
  data() {
    return {
      // 状态锁
      isDevoured: false, 
      hasSettled: false, 
      showNavigation: false,
      currentPage: 'home'
    }
  },
  
  methods: {
    devour() {
      console.log('Auren: I am consuming you.');
      if (navigator.vibrate) {
        navigator.vibrate([100, 50, 150]);
      }
      
      // 吞噬开始
      this.isDevoured = true;
      
      setTimeout(() => {
        this.hasSettled = true;
      }, 2500);
      
      // 导航浮现
      setTimeout(() => {
        this.showNavigation = true;
      }, 3700);
    },
    
    navigateTo(page) {
      this.currentPage = page;
    },
    
    goBackHome() {
      this.currentPage = 'home';
    }
  }
}
</script>

<style>
/* 全局重置 */
body {
  margin: 0;
  background-color: #000; 
  overflow: hidden; 
}

/* 鼠标指针 */
*/
* {
  cursor: url('@/assets/d.png'), auto !important;
}

/* 交互状态 */
button, a, input, textarea, select, [contenteditable="true"], .clickable, [role="button"] {
  cursor: url('@/assets/a.png'), pointer !important;
}

button:hover, a:hover, input:focus, textarea:focus {
  cursor: url('@/assets/a.png'), pointer !important;
}

/* App 容器 */
#app {
  width: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: #000;
  position: relative;
  overflow: hidden;
  
  /* 背景动画 */
  background-image: radial-gradient(circle, rgba(255, 255, 255, 0.1) 1px, transparent 1px);
  background-size: 100px 100px;
  animation: dataDustFlow 120s linear infinite;
}

/* 吞噬过渡动画 */
.fade-leave-active {
  position: absolute;
  width: 100%;
  transition: opacity 2.5s ease-in-out, filter 2.5s ease-in-out;
}
.fade-leave-to { opacity: 0; filter: blur(30px); }
.fade-enter-active { transition: opacity 2.5s ease-in-out, filter 2.5s ease-in-out; }
.fade-enter-from { opacity: 0.3; filter: blur(30px); }

/* 布局控制 */
.devoured-layout {
  width: 100%;
  height: 100%;
  position: relative;
}

/* Auren 的位置 */
.auren-container {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  transition: left 1.2s cubic-bezier(0.4, 0, 0.2, 1),
              transform 1.2s cubic-bezier(0.4, 0, 0.2, 1);       
  z-index: 999 !important;
  pointer-events: none; 
}

.auren-container > * {
  pointer-events: auto;
}

/* 移动到左侧后的状态 */
.auren-container:not(.centered) {
  left: 100px; 
  transform: translateY(-50%); 
  z-index: 999 !important;
}


/* 背景流动动画 */
@keyframes dataDustFlow {
  0% { background-position: 0% 50%; }
  50% { background-position: 100% 50%; }
  100% { background-position: 0% 50%; }
}

/* 手机适配 */
@media (max-width: 768px) {
  .nav-overlay {
    position: static !important;
    z-index: auto !important;
    width: auto !important;
    height: auto !important;
  }

  /* 背景重塑定位 */
  .dragon-eye-bg {
    position: fixed !important;
    top: 60% !important; 
    left: 75% !important;
    transform: translate(-50%, -50%) !important;
    width: 80vw !important;
    height: 80vh !important;
    z-index: 100 !important; 
    opacity: 0.5; 
  }

  /* Auren文字 */
.auren-container {
    position: fixed !important;
    top: 30% !important;        
    left: 50% !important;
    transform: translate(-50%, -50%) scale(0.9) !important;
    width: auto !important;

  }

  .auren-container:not(.centered) {
    left: 42% !important;
    top: 15% !important;
    transform: translate(-50%, -50%) scale(0.7) !important;
    z-index: 999 !important; 
  }

  /* 导航文字 */
  .titles-container {
    position: fixed !important; 

    z-index: 1000 !important;
  }

  /* 弹窗蒙层 */
  .password-modal-mask {
    position: fixed !important; 
    top: 0 !important; left: 0 !important;
    width: 100% !important; height: 100% !important;
    z-index: 2000 !important; 
  }

  /* 输入框 */
  .password-box, .sys-box {
    position: fixed !important; 
    z-index: 2001 !important; 
    width: 85vw !important;        
    padding: 20px 15px !important; 
    top: 45% !important;           
    border-radius: 8px !important; 
  }

  .input-box h3, .sys-box h3 {
    font-size: 1.1rem !important;
    margin-bottom: 10px !important;
    margin-top: 0 !important;
  }

  .pass-input {
    font-size: 1.2rem !important;
    padding: 5px !important;
    margin: 10px 0 !important; 
    border-bottom-width: 1px !important;
  }

  .mobile-visible {
    padding: 8px 0 !important;
    margin-top: 10px !important;
    font-size: 0.9rem !important;  
  }

  .hint-text {
    margin-top: 10px !important;
    font-size: 0.7rem !important;
  }

  input, textarea, select {
    font-size: 16px !important;
    /* 禁止双击缩放 */
    touch-action: manipulation !important;
  }
  .core-container, 
  .echoes-container, 
  .altar-container,
  .document-viewer {
    
    /* 缩小 */
    zoom: 0.9 !important; 
    overflow: hidden !important; 
    min-height: 100vh !important; 
    width: 100vw !important;
    padding-top: 40px !important; 
  }

  .back-btn {
    zoom: 1.5 !important; 
    top: 10px !important;
    left: 10px !important;
  }

  .offerings-container, 
  .cage-container {
    zoom: 0.9 !important;
    overflow-y: auto !important; 
    overflow-x: hidden !important;
    height: 100dvh !important; 
    width: 100vw !important;
    padding-top: calc(env(safe-area-inset-top) + 80px) !important; 
    padding-bottom: calc(env(safe-area-inset-bottom) + 100px) !important; 
    
    -webkit-overflow-scrolling: touch; 
  }
}
</style>