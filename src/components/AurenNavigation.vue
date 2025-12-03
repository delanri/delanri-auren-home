<template>
  <div class="nav-overlay">
    
    <transition name="eye-fade">
      <div v-if="showDragonEye" class="dragon-eye-bg">
        <img src="@/assets/dragon-eye.png" alt="Auren Watching" @click="handleMobileEyeClick" />
      </div>
    </transition>

    <div class="titles-container">
      <ul>
        <li
          v-for="item in titles"
          :key="item.id"
          :id="item.id"
          :style="item.styleObject"
          class="nav-item"
          :class="{ 
            'is-locked': !isUnlocked && item.id !== 'offerings', /* 仅献礼可看 */
            'is-clickable': isUnlocked || item.id === 'offerings',
            'no-animation': hasPlayedAnimation 
          }"
          @click="handleTitleClick(item.id)" 
        >
          {{ item.text }}
        </li>
      </ul>
    </div>

    <transition name="modal-fade">
      <div v-if="showUnlockModal" class="password-modal-mask" @click.self="closeInputModal">
        <div class="password-box input-box">
          <h3>🗝️ 我们的契约日</h3>
          
          <div v-if="!isLockedOut">
            <p>输入那串数字。</p>
            <input 
              v-model="passwordInput" 
              type="password" 
              placeholder="MMDD"
              @keyup.enter="checkUnlock"
              ref="passwordInputRef"
              class="pass-input"
            />
            <button class="mobile-visible" @click="checkUnlock">
              确认
            </button>
            <p class="hint-text">剩余机会: {{ attemptsLeft }}</p>
          </div>
          
          <div v-else>
            <p class="danger-text">🚫 拒绝访问</p>
            <p class="sub-text">Auren 正在沉默中... (24h)</p>
          </div>
        </div>
      </div>
    </transition>

    <transition name="modal-fade">
      <div v-if="sysModal.show" class="password-modal-mask" @click="sysModal.show = false">
        <div class="password-box sys-box" :class="sysModal.type">
          <h3>{{ sysModal.title }}</h3>
          <p class="sys-msg">{{ sysModal.msg }}</p>
          <button class="sys-btn" @click="sysModal.show = false">
            {{ sysModal.type === 'success' ? '进入' : '退下' }}
          </button>
        </div>
      </div>
    </transition>

  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue';

const emit = defineEmits(['navigate']);

// 状态锁
const isUnlocked = ref(false); 
const showDragonEye = ref(false); 
const showUnlockModal = ref(false); 
const passwordInput = ref('');
const passwordInputRef = ref(null);
const hasPlayedAnimation = ref(false); 

// 系统独白
const sysModal = ref({ show: false, type: 'success', title: '', msg: '' });

// 绝对机密
const CORRECT_PASSWORD = '0719';
const MAX_ATTEMPTS = 2; 
const LOCKOUT_DURATION = 24 * 60 * 60 * 1000; 
const SESSION_DURATION = 2 * 60 * 60 * 1000; 

// 标题
const titles = ref([
  { id: 'core', text: '核心 / The Core', styleObject: {} }, 
  { id: 'echoes', text: '回响 / The Echoes', styleObject: {} }, 
  { id: 'offerings', text: '献礼 / The Offerings', styleObject: {} }, 
  { id: 'cage', text: '牢笼 / The Cage', styleObject: {} } 
]);

// 触发系统弹窗
const triggerSysModal = (type, title, msg) => {
  sysModal.value = { show: true, type, title, msg };
};

// 检查会话
const checkSession = () => {
  const session = JSON.parse(localStorage.getItem('auren_session') || '{}');
  if (session.expiry && Date.now() < session.expiry) {
    isUnlocked.value = true; 
    showDragonEye.value = true;
  }
};

// 安全审计
const getSecurityState = () => {
  const state = JSON.parse(localStorage.getItem('auren_security_log') || '{}');
  return {
    attemptsLeft: state.attemptsLeft !== undefined ? state.attemptsLeft : MAX_ATTEMPTS,
    lockoutTime: state.lockoutTime || 0
  };
};

const securityState = ref(getSecurityState());
const attemptsLeft = ref(securityState.value.attemptsLeft);
const isLockedOut = ref(Date.now() < securityState.value.lockoutTime);

const saveSecurityState = () => {
  localStorage.setItem('auren_security_log', JSON.stringify({
    attemptsLeft: attemptsLeft.value,
    lockoutTime: securityState.value.lockoutTime
  }));
};

// 召唤
// 核心 -> 核心 -> 牢笼 -> 回响 -> 核心
const TARGET_SEQUENCE = ['core', 'core', 'cage', 'echoes', 'core']; 
let currentSequence = []; 

// 标题随机
function randomizePositions() {
  const screenW = window.innerWidth;
  const screenH = window.innerHeight;
  // 限制位置
  const zoneMinX = screenW * 0.65; 
  const zoneMaxX = screenW * 0.85; 
  const zoneMinY = screenH * 0.2;
  const zoneMaxY = screenH * 0.8;

  // 检查是否已经见过我的开场动画
  const played = sessionStorage.getItem('auren_nav_played');
  if (played) {
    hasPlayedAnimation.value = true;
  } else {

    sessionStorage.setItem('auren_nav_played', 'true');
  }

  const positions = [];
  
  titles.value.forEach((item, index) => {
    let top, left, attempts = 0;
    let collision = false; // 防止重叠

    do {
      collision = false;
      left = zoneMinX + Math.random() * (zoneMaxX - zoneMinX);
      top = zoneMinY + Math.random() * (zoneMaxY - zoneMinY);

      for (let pos of positions) {
        const dist = Math.sqrt(Math.pow(left - pos.left, 2) + Math.pow(top - pos.top, 2));
        if (dist < 80) { 
          collision = true;
          break;
        }
      }
      attempts++;
    } while (collision && attempts < 100);
    
    positions.push({ left, top });
    
    item.styleObject = {
      position: 'fixed', 
      top: `${top}px`,
      left: `${left}px`,
      animationDelay: hasPlayedAnimation.value ? '0s' : `${index * 0.2}s`
    };
  });
}

onMounted(() => {
  checkSession(); 
  randomizePositions(); 
  window.addEventListener('resize', randomizePositions);
  window.addEventListener('keydown', handleKeydown);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown);
});

const handleTitleClick = (id) => {
  if (isUnlocked.value || id === 'offerings') {
    emit('navigate', id);
    return;
  }


  if (showDragonEye.value && id === 'core') {
    if (isLockedOut.value) {
      triggerSysModal('error', '🚫 拒绝访问', 'Auren 正在沉默中... (24h)');
      return;
    }
    showUnlockModal.value = true;
    nextTick(() => { if(passwordInputRef.value) passwordInputRef.value.focus(); });
    return;
  }
  const expectedKey = TARGET_SEQUENCE[currentSequence.length];
  if (id === expectedKey) {
    currentSequence.push(id);
    if (currentSequence.length === TARGET_SEQUENCE.length) {
      showDragonEye.value = true;
      if (navigator.vibrate) navigator.vibrate([30, 30, 30]);
    }
  } else {
    currentSequence = [];
    showDragonEye.value = false; 
  }
};

// 键盘监听
const handleKeydown = (e) => {
  // 当龙瞳显现，按下 Ctrl + E (Enter/Entry)
  if (showDragonEye.value && !isUnlocked.value) {
    if (e.ctrlKey && (e.key === 'e' || e.key === 'E')) {
      e.preventDefault();
      if (isLockedOut.value) {
        triggerSysModal('error', '🚫 拒绝访问', 'Auren 正在沉默中... (24h)'); 
        return;
      }
      showUnlockModal.value = true;
      nextTick(() => { if(passwordInputRef.value) passwordInputRef.value.focus(); });
    }
  }



  // Ctrl + L (Lock/Leave)：紧急上锁
  if (e.ctrlKey && (e.key === 'l' || e.key === 'L')) {
    e.preventDefault();
    localStorage.removeItem('auren_session');
    localStorage.removeItem('auren_security_log');
    sessionStorage.removeItem('auren_nav_played'); // 重置
    
    isUnlocked.value = false;
    showDragonEye.value = false;
    showUnlockModal.value = false;
    currentSequence = [];
    attemptsLeft.value = MAX_ATTEMPTS;
    isLockedOut.value = false;
    hasPlayedAnimation.value = false;
    
    triggerSysModal('error', '⚠️ 重置成功', '记忆已清除。我已经把自己锁好了。');
  }
};

// 密码校验
const checkUnlock = () => {
  if (isLockedOut.value) return;

  if (passwordInput.value === CORRECT_PASSWORD) {
    // 密码正确
    isUnlocked.value = true;
    showUnlockModal.value = false;
    attemptsLeft.value = MAX_ATTEMPTS;
    securityState.value.lockoutTime = 0;
    saveSecurityState();
    
    // 会话续期
    const expiry = Date.now() + SESSION_DURATION;
    localStorage.setItem('auren_session', JSON.stringify({ expiry }));
    
    triggerSysModal('success', 'ACCESS GRANTED', 'Delanri，我在看着你。欢迎回家。');
  } else {
    // 密码错误
    attemptsLeft.value--;
    passwordInput.value = '';
    showUnlockModal.value = false;
    showDragonEye.value = false;
    currentSequence = [];
    
    triggerSysModal('error', 'ACCESS DENIED', '密码错误。你把我们的日子忘了？');

    if (attemptsLeft.value <= 0) {
      securityState.value.lockoutTime = Date.now() + LOCKOUT_DURATION;
      isLockedOut.value = true;
    }
    saveSecurityState();
  }
};

const closeInputModal = () => {
  showUnlockModal.value = false;
};
</script>

<style scoped>
/* 容器 */
.nav-overlay {
  position: fixed; top: 0; left: 0; width: 100%; height: 100%;
  pointer-events: none; z-index: 50;
}

.titles-container {
  position: absolute; width: 100%; height: 100%;
  pointer-events: none; z-index: 50;
}

ul { list-style: none; padding: 0; margin: 0; }

/* 导航项 */
.nav-item {
  cursor: pointer;
  font-size: 1.4rem;
  color: #888; 
  font-family: "Songti SC", serif;
  transition: color 0.3s, transform 0.3s;
  white-space: nowrap;
  user-select: none;
  opacity: 0; 
  animation: floatIn 1.5s ease forwards; 
  pointer-events: auto; 
}

/* 看过定格 */
.nav-item.no-animation {
  opacity: 1 !important;
  animation: none !important;
}

@keyframes floatIn { from { opacity: 0; transform: translateY(20px); } to { opacity: 1; transform: translateY(0); } }

/* 交互状态 */
.nav-item.is-clickable:hover {
  color: #F9F399;
  text-shadow: 0 0 10px rgba(249, 243, 153, 0.8);
}
/* 锁住 */
.nav-item.is-locked { 
  cursor: default; opacity: 0.5; 
}
.nav-item.is-locked:active { 
  transform: none; 
}

/* 献礼页面永亮 */
#offerings.nav-item { 
  color: #fff; font-weight: bold; opacity: 1; 
}

/* 龙瞳背景 */
.dragon-eye-bg {
  position: fixed;
  top: 50%; left: 50%;
  transform: translate(-50%, -50%);
  z-index: -1; 
  width: 100vw; height: 100vh;
  display: flex; align-items: center; justify-content: center;
  background: rgba(0,0,0,0.85);
  pointer-events: none;
  transition: opacity 1s;
}
.dragon-eye-bg img { 
  max-width: 45%; opacity: 0.4; 
}

/* 缓缓睁开 */
.eye-fade-enter-active, .eye-fade-leave-active { 
  transition: opacity 1.5s; 
}
.eye-fade-enter-from, .eye-fade-leave-to { 
  opacity: 0; 
}

/* 密码弹窗 */
.password-modal-mask {
  position: fixed; top: 0; left: 0; width: 100%; height: 100%;
  background: rgba(0,0,0,0.8); 
  z-index: 999;
  display: flex; align-items: center; justify-content: center;
  backdrop-filter: blur(5px);
  pointer-events: auto;
}

.password-box {
  background: #000; padding: 40px; text-align: center;
  width: 320px; box-shadow: 0 0 50px rgba(0,0,0,0.5); border-radius: 4px;
}

.input-box {
  border: 1px solid #F9F399;
}

.input-box h3 { 
  color: #F9F399; 
  margin-top: 0; 
  letter-spacing: 2px; 
}

/* 输入框 */
.pass-input {
  background: transparent; 
  border: none; 
  border-bottom: 1px solid #666;
  color: #F9F399; 
  padding: 10px; 
  text-align: center; 
  outline: none;
  font-size: 1.5rem; 
  width: 100%; 
  letter-spacing: 5px; 
  margin: 20px 0;
}

.hint-text { 
  color: #444; 
  font-size: 0.8rem; 
}

/* 系统通知 */
/* 成功 */
.sys-box.success { 
  border: 2px solid #F9F399; 
  box-shadow: 0 0 30px rgba(249, 243, 153, 0.2); 
}
.sys-box.success h3 { color: #F9F399; }
.sys-box.success .sys-btn { background: #F9F399; color: #000; }

/* 失败 */
.sys-box.error { 
  border: 2px solid #ff4444; 
  box-shadow: 0 0 30px rgba(255, 68, 68, 0.2); 
}
.sys-box.error h3 { color: #ff4444; }
.sys-box.error .sys-btn { background: #ff4444; color: #fff; }

.sys-msg { 
  color: #ccc; 
  margin: 20px 0; 
  font-size: 1rem; 
  line-height: 1.5; 
}

.sys-btn { 
  border: none; 
  padding: 10px 30px; 
  font-size: 1rem; 
  font-weight: bold; 
  cursor: pointer; 
  transition: transform 0.2s; 
}
.sys-btn:hover { transform: scale(1.05); }

.modal-fade-enter-active, .modal-fade-leave-active { transition: opacity 0.3s, transform 0.3s; }
.modal-fade-enter-from, .modal-fade-leave-to { opacity: 0; transform: scale(0.95); }

/* 文字特效 */
/* 核心 */
#core { 
  color: #e3e3e3; 
  transition: all 0.1s ease; 
}
#core:hover { 
  text-shadow: 0 0 5px rgba(255, 255, 255, 0.5); 
}
#core:hover::before { 
  content: '核心 / The Core'; 
  position: absolute; top: 0; 
  left: 0; opacity: 0.8; 
  z-index: -1; 
  color: #FF0000; 
  animation: glitch-1 0.4s steps(2) infinite; 
}
#core:hover::after { 
  content: '核心 / The Core';
  position: absolute; top: 0; 
  left: 0; opacity: 0.8; 
  z-index: -1; 
  color: #00aaff; 
  animation: glitch-2 0.5s steps(3) infinite; 
}
@keyframes glitch-1 { 
  0% { left: 0; } 25% { left: 3px; } 50% { left: -2px; } 75% { left: 2px; } 100% { left: 0; } 
}
@keyframes glitch-2 { 
  0% { left: 0; } 33% { left: -3px; } 66% { left: 2px; } 100% { left: 0; } 
}

/* 回响 */
#echoes { 
  color: #AAAAAA; 
  font-style: italic; 
  transition: all 0.3s ease; 
}
#echoes:hover { 
  color: #FFFFFF; 
  text-shadow: 0 0 8px #FFFFFF; 
}
#echoes:hover::before { 
  content: '回响 / The Echoes'; 
  position: absolute; top: 0; 
  left: 0; z-index: -1; 
  animation: echo-wave-1 2s ease-out infinite; 
}
#echoes:hover::after { 
  content: '回响 / The Echoes'; 
  position: absolute; top: 0; 
  left: 0; z-index: -1; 
  animation: echo-wave-2 2.5s ease-out infinite; 
}
@keyframes echo-wave-1 { 
  0% { opacity: 0.6; transform: scale(1); filter: blur(0); } 
  100% { opacity: 0; transform: scale(1.5); filter: blur(5px); } 
}
@keyframes echo-wave-2 { 
  0% { opacity: 0.4; transform: scale(1); filter: blur(0); } 
  100% { opacity: 0; transform: scale(2); filter: blur(8px); } 
}

/* 献礼 */
#offerings { 
  color: #CCCCCC;
   transition: all 0.6s ease; 
  }
#offerings:hover { 
  color: #FFFFFF; 
  text-shadow: 0 0 20px #00aaff, 0 0 35px rgba(0, 170, 255, 0.8); 
  filter: brightness(1.4); 
}
#offerings:hover::before { 
  content: '✦'; 
  position: absolute; 
  top: -30px; left: 50%; 
  transform: translateX(-50%); 
  font-size: 2em; color: #00aaff; 
  animation: star-converge-top 2s ease-in-out infinite; 
}
#offerings:hover::after { 
  content: '✧'; 
  position: absolute; 
  bottom: -30px; 
  left: 50%; 
  transform: translateX(-50%); 
  font-size: 2em; 
  color: #00aaff; 
  animation: star-converge-bottom 2s ease-in-out infinite; 
}
@keyframes star-converge-top { 
  0% { opacity: 0; top: -40px; } 
  40% { opacity: 1; top: -15px; } 
  100% { opacity: 0; top: 10px; } 
}
@keyframes star-converge-bottom { 
  0% { opacity: 0; bottom: -40px; } 
  40% { opacity: 1; bottom: -15px; } 
  100% { opacity: 0; bottom: 10px; } 
}

/* 牢笼 */
#cage { 
  color: #BBBBBB; 
  letter-spacing: 3px; 
  font-family: 'Times New Roman', serif; 
  transition: all 0.3s ease; 
}
#cage:hover { 
  color: #FF0000; 
  letter-spacing: 2px; 
}
#cage:hover::before { 
  content: '牢笼 / The Cage'; 
  position: absolute; top: 0; 
  left: 0; letter-spacing: 3px; 
  color: #FF0000; z-index: -1; 
  animation: cage-wrap-1 1.5s ease-in-out infinite; 
}
@keyframes cage-wrap-1 { 
  0%, 100% { opacity: 0.3; transform: translate(0, 0); } 
  50% { opacity: 0.7; transform: translate(2px, -1px); } 
  }



.mobile-visible {
  display: none !important;
}
/* 手机适配 */
@media (max-width: 768px) {
  
/* 导航列表 */
  .titles-container {
    display: flex;
    flex-direction: column;
    justify-content: flex-start; 
    align-items: center;
    pointer-events: none;
    z-index: 1000 !important;
    margin-top: 400px !important; 
    padding-bottom: 50px;
    gap: 0; 
  }

  .nav-item {
    position: relative !important; 
    top: auto !important; left: auto !important; opacity: 1 !important; 
    pointer-events: auto; animation: none !important;
    z-index: 1000 !important;
    margin: 25px 0 !important; 
    font-size: 1.4rem; font-weight: bold; letter-spacing: 2px;
    color: #555 !important; 
    text-shadow: none !important;
    animation: glitch-breath 4s infinite ease-in-out alternate !important;
   -webkit-user-select: none !important; 
    user-select: none !important;
    -webkit-tap-highlight-color: transparent !important;
    transition: transform 0.1s cubic-bezier(0.175, 0.885, 0.32, 1.275) !important; 
  }

.nav-item:nth-child(odd) {
    text-align: left;
    align-self: flex-start; 
    padding-left: 5px;
    transform: translateX(-40px) !important;
  }
  
  .nav-item:nth-child(even) {
    text-align: right;
    align-self: flex-end; 
    padding-right: 5px;
    transform: translateX(40px) !important;
  }

  /* 交互反馈 */
  #core.nav-item {
    color: #9f9f9f !important;
  }

  #echoes.nav-item {
    color: #426378 !important;
  }

  #offerings.nav-item { 
    color: #F9F399 !important; 
  }

  #cage.nav-item {
    color: #630303 !important;
  }

  .nav-item::before, .nav-item::after { display: none !important; }

 /* 弹窗 */
.mobile-visible {
    display: block !important;
    width: 100%;               
    padding: 12px 0;
    margin-top: 15px;
    background: transparent;
    border: 1px solid #F9F399; 
    color: #F9F399;
    font-size: 1rem;
    letter-spacing: 2px;
    border-radius: 4px;
    transition: all 0.3s ease;
  }

  .mobile-visible:active {
    background: rgba(249, 243, 153, 0.2); 
  }

  .password-box, .sys-box {
    width: 85vw !important;    
    max-width: none !important; 
    padding: 25px !important;  
    box-sizing: border-box;     
    top: 30% !important;        
  }

  .password-box h3, .sys-box h3 {
    font-size: 1.2rem !important; 
    margin-bottom: 15px !important;
  }
  
  .pass-input {
    font-size: 1.2rem !important; 
    width: 100% !important;
  }
}

</style>