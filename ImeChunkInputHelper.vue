<script setup lang="ts">
/**
 * =====================================================================================
 * CJK IME (한글/중국어/일어) 입력 중복 방지 및 모바일 웹 터미널을 위한 
 * Decoupled Chunk Input Helper Component (Vue 3 + TypeScript)
 * =====================================================================================
 * 
 * 💡 [핵심 아키텍처 및 해결하는 문제]
 * 1. xterm.js 네이티브 Composition 버그 예방:
 *    xterm.js 가상 터미널 환경에 직접 CJK 문자를 타이핑할 경우, 브라우저 IME 조합기(Composition)와 
 *    터미널의 즉각적인 Stdin 에코 백엔드가 충돌하여 글자가 중복 복제되어 타이핑되는 현상("아" + "ㅇ" -> "아아ㅇ")이 빈발합니다.
 * 
 * 2. 입력 버퍼 디커플링 (Decoupled Sandbox):
 *    가상 터미널로의 직접 키 입력을 방지하고, 표준 HTML Input 요소를 샌드박스로 두어
 *    사용자의 한글 타이핑이 브라우저 순수 완성형 엔진 안에서 안전하게 종결되도록 유도합니다.
 * 
 * 3. 청크 스트리밍 (Chunk Injection):
 *    완성된 문장이나 명령 구문을 하나의 단위(Chunk)로 결합하여 단 한 번의 에코 스트림(onData)으로 
 *    PTY(Pseudoterminal) 백엔드로 쏴주어, 자음/모음이 쪼개지거나 깨지는 불상사를 차단합니다.
 */

import { ref, onMounted, onUnmounted } from "vue";

// 컴포넌트 인터페이스 정의
interface Props {
  // 사용 중인 xterm.js Terminal 인스턴스 (부모로부터 주입 받거나 전송 콜백만으로도 구동 가능)
  terminalInstance?: any; 
  // 터미널 백엔드 Websocket 또는 PTY Stdin 송신용 콜백 함수
  onSendStdin: (data: string) => void;
}

const props = withDefaults(defineProps<Props>(), {
  terminalInstance: null
});

// 반응형 상태 변수
const chunkInputValue = ref("");
const showChunkInput = ref(true);
const autoEnter = ref(true); // 엔터 시 자동 개행(
) 추가 여부
const isMobileDevice = ref(false);

// 모바일 가상 키보드 대응을 위한 비주얼 뷰포트 높이
const viewportHeight = ref("auto");

// IME 청크 입력기 입력창 포커스 함수
const focusInput = () => {
  const inputEl = document.querySelector(".chunk-terminal-input") as HTMLInputElement;
  if (inputEl) {
    inputEl.focus();
  }
};

// 1. 청크 스트림 전송 핵심 함수 (The Weapon)
const sendChunk = () => {
  const text = chunkInputValue.value;
  if (!text) return;

  // 자동 개행(
 = Carriage Return) 옵션 활성화 시 끝에 개행 추가
  const dataToSend = text + (autoEnter.value ? "\r" : "");
  
  // PTY 송신 콜백 실행
  props.onSendStdin(dataToSend);

  // 입력 버퍼 청소
  chunkInputValue.value = "";

  // 전송 즉시 모바일 소프트 키보드가 닫히지 않도록 포커스 복원 유지
  setTimeout(() => {
    focusInput();
  }, 20);
};

// 2. 특수 제어키 및 헬퍼 단축키 수동 인젝션 함수
const sendHelperKey = (keyType: string) => {
  let controlCode = "";
  switch (keyType) {
    case "tab": controlCode = "\t"; break;
    case "esc": controlCode = "\x1b"; break;
    case "ctrlc": controlCode = "\x03"; break; // SIGINT 인터럽트
    case "ctrld": controlCode = "\x04"; break; // EOF 전송
    case "ctrlk": controlCode = "\x0b"; break; // 라인 삭제
    case "enter": controlCode = "\r"; break;
    case "up": controlCode = "\x1b[A"; break;    // 역사 히스토리 백 tracking
    case "down": controlCode = "\x1b[B"; break;
    case "left": controlCode = "\x1b[D"; break;
    case "right": controlCode = "\x1b[C"; break;
  }

  if (controlCode) {
    props.onSendStdin(controlCode);
  }
};

// 3. CTRL + 알파벳 형태의 커스텀 제어 조합 생성기
const sendCtrlCombo = (char: string) => {
  if (!char) return;
  const targetChar = char.toUpperCase().trim();
  const charCode = targetChar.charCodeAt(0);
  
  if (charCode >= 65 && charCode <= 90) {
    const controlCode = String.fromCharCode(charCode - 64); // ASCII Control Char offset
    props.onSendStdin(controlCode);
    console.log(`[IME Controller] Sent CTRL+${targetChar} (${charCode - 64}) sequence`);
  }
};

// 4. 모바일 레이아웃 및 뷰포트 높이 보정 로직
const handleViewportChange = () => {
  if (window.visualViewport) {
    viewportHeight.value = `${window.visualViewport.height}px`;
    // 부모로부터 주입된 터미널 인스턴스가 있다면 dynamic fit/refit 호출 유도
    if (props.terminalInstance && props.terminalInstance.fit) {
      try {
        props.terminalInstance.fit();
      } catch (err) {
        console.warn("Terminal refit failed on viewport resize:", err);
      }
    }
  }
};

// 스냅백 스크롤 방지 (모바일 가상 키보드 아웃 시 여백 제거)
const handleInputBlur = () => {
  if (isMobileDevice.value) {
    setTimeout(() => {
      window.scrollTo({ top: 0, left: 0, behavior: "instant" });
      handleViewportChange();
    }, 80);
  }
};

onMounted(() => {
  // 모바일/태블릿 탐지 정밀 표현식
  isMobileDevice.value = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent);

  if (window.visualViewport) {
    window.visualViewport.addEventListener("resize", handleViewportChange);
    window.visualViewport.addEventListener("scroll", handleViewportChange);
  }
  handleViewportChange();
});

onUnmounted(() => {
  if (window.visualViewport) {
    window.visualViewport.removeEventListener("resize", handleViewportChange);
    window.visualViewport.removeEventListener("scroll", handleViewportChange);
  }
});
</script>

<template>
  <div class="ime-chunk-controller font-mono" :class="{ 'is-mobile': isMobileDevice }">
    <!-- 컨트롤러 토글 트리거 바 (터미널 하단에 컴팩트하게 노출) -->
    <div class="control-header flex items-center justify-between">
      <div class="flex items-center gap-2">
        <span class="status-indicator active"></span>
        <span class="title">CJK IME BUFFER CONTROLLER</span>
      </div>
      
      <div class="flex items-center gap-1.5">
        <!-- 자동 엔터 스위치 -->
        <label class="auto-enter-label flex items-center gap-1">
          <input type="checkbox" v-model="autoEnter" />
          <span>AUTO ENTER</span>
        </label>
        
        <button 
          @click="showChunkInput = !showChunkInput" 
          class="toggle-btn"
          :class="{ 'active': showChunkInput }"
        >
          ⌨️ {{ showChunkInput ? 'HIDE' : 'SHOW' }}
        </button>
      </div>
    </div>

    <!-- 청크 입력 메인 패널 -->
    <div v-if="showChunkInput" class="control-body flex flex-col gap-2">
      <!-- 1층: 특수 키패드 및 D-Pad (모바일 원터치 대응 및 특수 제어키 매핑) -->
      <div class="flex gap-2 items-stretch">
        <div class="helper-grid flex-1 flex flex-col gap-1">
          <!-- 기능 제어 Row 1 -->
          <div class="flex gap-1">
            <button @click="sendHelperKey('tab')" class="action-btn">TAB</button>
            <button @click="sendHelperKey('esc')" class="action-btn">ESC</button>
            <button @click="sendHelperKey('ctrlc')" class="action-btn danger">CTRL+C</button>
          </div>
          <!-- 기능 제어 Row 2 -->
          <div class="flex gap-1">
            <button @click="sendHelperKey('ctrld')" class="action-btn danger">CTRL+D</button>
            <button @click="sendHelperKey('ctrlk')" class="action-btn">CTRL+K</button>
            <!-- 커스텀 CTRL 조합 팝업 대용 원터치 trigger -->
            <button @click="sendCtrlCombo('Z')" class="action-btn">CTRL+Z</button>
          </div>
        </div>

        <!-- 초대형 다이렉트 엔터키 (↵) -->
        <button @click="sendHelperKey('enter')" class="giant-enter-btn">
          ↵
        </button>

        <!-- D-PAD 방향키 그리드 -->
        <div class="dpad-grid grid grid-cols-3 grid-rows-2 gap-1">
          <button @click="props.onSendStdin('/')" class="action-btn">/</button>
          <button @click="sendHelperKey('up')" class="action-btn">▲</button>
          <div class="spacer"></div>
          
          <button @click="sendHelperKey('left')" class="action-btn">◀</button>
          <button @click="sendHelperKey('down')" class="action-btn">▼</button>
          <button @click="sendHelperKey('right')" class="action-btn">▶</button>
        </div>
      </div>

      <!-- 2층: 청크 인풋 전송 샌드박스 (IME의 철옹성) -->
      <div class="input-row flex items-center gap-1.5">
        <input 
          v-model="chunkInputValue"
          @keydown.enter.stop.prevent="sendChunk"
          @blur="handleInputBlur"
          type="text"
          placeholder="한글/영문 명령어를 입력한 뒤 ENTER를 누르세요..."
          class="chunk-terminal-input flex-1"
        />
        <button @click="sendChunk" class="send-btn">
          SEND
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* 
 * Vibrant Neon Cyberpunk Aesthetic 
 * 가상 터미널 환경에 적합한 다크 모드 및 형광 그린/네온 사이언 테마
 */
.ime-chunk-controller {
  background: rgba(10, 10, 10, 0.95);
  border: 1px solid rgba(57, 255, 20, 0.25);
  border-radius: 4px;
  padding: 8px;
  box-sizing: border-box;
  color: #39ff14;
  box-shadow: 0 0 15px rgba(57, 255, 20, 0.05);
}

.control-header {
  font-size: 10px;
  font-weight: bold;
  letter-spacing: 1px;
  margin-bottom: 6px;
  border-bottom: 1px solid rgba(57, 255, 20, 0.15);
  padding-bottom: 4px;
}

.status-indicator {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  display: inline-block;
}
.status-indicator.active {
  background: #39ff14;
  box-shadow: 0 0 8px #39ff14;
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0% { opacity: 0.4; }
  50% { opacity: 1; }
  100% { opacity: 0.4; }
}

.title {
  text-shadow: 0 0 5px rgba(57, 255, 20, 0.4);
}

.auto-enter-label {
  font-size: 8px;
  cursor: pointer;
  color: rgba(57, 255, 20, 0.7);
  user-select: none;
}
.auto-enter-label input[type="checkbox"] {
  accent-color: #39ff14;
  cursor: pointer;
}

.toggle-btn {
  background: transparent;
  border: 1px solid rgba(57, 255, 20, 0.4);
  color: #39ff14;
  font-size: 8px;
  padding: 2px 6px;
  border-radius: 2px;
  cursor: pointer;
  font-family: inherit;
  transition: all 0.2s;
}
.toggle-btn:hover, .toggle-btn.active {
  background: #39ff14;
  color: #000;
  font-weight: bold;
}

/* 바디 요소 */
.helper-grid {
  min-width: 140px;
}

.action-btn {
  flex: 1;
  background: rgba(20, 20, 20, 0.8);
  border: 1px solid rgba(57, 255, 20, 0.3);
  color: #39ff14;
  font-size: 9px;
  font-weight: bold;
  padding: 6px 0;
  border-radius: 2px;
  cursor: pointer;
  font-family: inherit;
  transition: all 0.15s;
  text-align: center;
}
.action-btn:hover {
  background: rgba(57, 255, 20, 0.15);
  box-shadow: 0 0 6px rgba(57, 255, 20, 0.2);
}
.action-btn.danger {
  color: #ff5555;
  border-color: rgba(255, 85, 85, 0.3);
}
.action-btn.danger:hover {
  background: rgba(255, 85, 85, 0.15);
  box-shadow: 0 0 6px rgba(255, 85, 85, 0.2);
}

.giant-enter-btn {
  width: 36px;
  background: rgba(0, 191, 255, 0.1);
  border: 1px solid rgba(0, 191, 255, 0.4);
  color: #00bfff;
  font-size: 18px;
  font-weight: 900;
  border-radius: 2px;
  cursor: pointer;
  font-family: inherit;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
}
.giant-enter-btn:hover {
  background: rgba(0, 191, 255, 0.25);
  box-shadow: 0 0 8px rgba(0, 191, 255, 0.3);
}

.dpad-grid {
  width: 90px;
}
.spacer {
  pointer-events: none;
  opacity: 0;
}

/* 입력 필드 Row */
.input-row {
  margin-top: 4px;
}

.chunk-terminal-input {
  background: rgba(0, 0, 0, 0.9);
  border: 1px solid rgba(57, 255, 20, 0.4);
  color: #39ff14;
  font-family: inherit;
  font-size: 11px;
  padding: 6px 10px;
  border-radius: 3px;
  transition: all 0.2s;
}
.chunk-terminal-input:focus {
  outline: none;
  border-color: #39ff14;
  box-shadow: 0 0 8px rgba(57, 255, 20, 0.3);
}
.chunk-terminal-input::placeholder {
  color: rgba(57, 255, 20, 0.35);
  font-size: 10px;
}

.send-btn {
  background: transparent;
  border: 1px solid rgba(57, 255, 20, 0.5);
  color: #39ff14;
  font-size: 10px;
  font-weight: bold;
  padding: 6px 12px;
  border-radius: 3px;
  cursor: pointer;
  font-family: inherit;
  transition: all 0.2s;
}
.send-btn:hover {
  background: #39ff14;
  color: #000;
  box-shadow: 0 0 10px rgba(57, 255, 20, 0.4);
}

/* 모바일 전용 미세 튜닝 */
.is-mobile {
  border-radius: 0;
  border-left: none;
  border-right: none;
}
</style>
