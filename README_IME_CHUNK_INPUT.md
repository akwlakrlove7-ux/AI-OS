# 🛡️ CJK IME 한글 중복 입력 해결사: Decoupled Chunk Input Pattern

웹 기반 가상 터미널(xterm.js 등) 및 실시간 WebSocket 동시성 환경에서 **한글, 중국어, 일어(CJK) 등 결합형 문자셋(IME)** 입력 시 발생하는 고질적인 글자 복제 현상을 완벽히 해결하는 핵심 아키텍처 가이드입니다.

---

## 1. 겪고 계신 문제의 본질 (Why is this happening?)

xterm.js나 일반 소켓 입력 버퍼에 한글을 직접 타이핑할 때 **"아" 입력 후 "ㅇ"을 누르면 "아아ㅇ"으로 복제되거나, 글자가 무작위로 깨져서 들어가는 현상**은 아래와 같은 아키텍처적 불일치로 발생합니다.

```
[사용자 키보드] ──> [브라우저 한글 결합기(Composition)]
                           │  (초성 + 중성 + 종성 조합 진행 중...)
                           ▼
                  [xterm.js Key Listener] 
                           │  (조합 도중의 미완성 키 입력을 소켓으로 즉시 인젝션!)
                           ▼
                 [PTY Host (stdin)] ──> [Terminal Echo-back Stream]
                                                    │
                                                    ▼
                                           [xterm.js Viewport]
```

* **원인**: xterm.js의 네이티브 키 리스너는 글자가 완전히 완성되기 전에 브라우저의 `compositionupdate` 이벤트를 가로채거나 비동기적으로 문자를 백엔드 PTY로 에코 전송합니다. 이로 인해 완성된 한글이 넘어가는 타이밍과 조합 중인 낱자가 넘어가는 타이밍이 꼬이면서 입력 스트림에 문자가 복사/중복 입력됩니다.

---

## 2. 해결책: 데드락을 방지하는 "청크 입력기" 아키텍처 (Decoupled Sandboxing)

이 문제를 해결하기 위한 가장 우아하고 실무적인 해법은 **터미널 화면과 입력 영역을 물리적으로 완전 디커플링(Decoupling)하고, 완성된 글자 덩어리(Chunk)를 일괄 스트림으로 송신**하는 것입니다.

```
┌──────────────────────────────────────────────────────────────┐
│                  [ 가상 터미널 Viewport ]                    │
│   (Read-Only 형태로 작동: 직접 타이핑 및 포커스 차단)        │
└──────────────────────────────┬───────────────────────────────┘
                               │ (Stdin single-stream 전송)
                               ▼
┌──────────────────────────────────────────────────────────────┐
│        [ IME 샌드박스 청크 입력기 (Chunk Input) ]            │
│  - 독립된 HTML5 표준 Input 박스에서 안전하게 한글 조합       │
│  - 자동 개행(\r)과 함께 명령 단위(Chunk)로 단 한 번에 주입   │
└──────────────────────────────────────────────────────────────┘
```

1. **IME 조합 샌드박싱**: 사용자는 터미널 뷰포트가 아닌, 컴포넌트 하단에 격리된 표준 HTML `<input>` 창에 타이핑합니다. 브라우저의 OS 표준 IME 엔진이 이 샌드박스 내부에서 완벽하게 한글을 조합하므로 터미널 간섭이 0%가 됩니다.
2. **청크 인젝션 (Single-Stream Chunk Injection)**: 단어나 전체 명령을 완성한 뒤 `Send` 버튼이나 `Enter` 키를 탭하는 순간, 모아서 완성한 텍스트 한 덩어리(`Chunk`)를 단 한 줄의 `onData` 스트림으로 PTY 소켓에 직접 밀어 넣습니다.
3. **모바일 키보드 최적화**: 모바일 소프트 키보드가 올라오며 전체 레이아웃이 찌그러지거나 아래 글자가 잘리는 현상을 감지(`visualViewport.height`)하여 가상 터미널을 다이내믹하게 `fit()` 하고 화면 스냅백을 자동으로 제어합니다.

---

## 3. 이식 및 활용 방법 (How to Integrate)

바탕화면에 저장된 `ImeChunkInputHelper.vue` 컴포넌트는 모든 Vue 3 + TypeScript 가상 터미널 환경에 즉시 적용할 수 있도록 설계되었습니다.

### A. 기본적인 컴포넌트 연동 예제

```html
<script setup lang="ts">
import { ref, onMounted } from "vue";
import { Terminal } from "xterm";
import { FitAddon } from "xterm-addon-fit";
import ImeChunkInputHelper from "./ImeChunkInputHelper.vue";

const terminalContainer = ref<HTMLElement | null>(null);
const term = ref<Terminal | null>(null);
const fitAddon = new FitAddon();
const socket = ref<WebSocket | null>(null);

// 1. 소켓 및 터미널 초기화
onMounted(() => {
  term.value = new Terminal({ cursorBlink: true });
  term.value.loadAddon(fitAddon);
  term.value.open(terminalContainer.value!);
  fitAddon.fit();

  socket.value = new WebSocket("ws://localhost:8000/ws/terminal");
  
  socket.value.onmessage = (event) => {
    term.value?.write(event.data);
  };
});

// 2. [The Key] 청크 입력기 전송 핸들러 바인딩
const handleSendStdin = (data: string) => {
  if (socket.value && socket.value.readyState === WebSocket.OPEN) {
    // 백엔드 PTY 소켓으로 일괄 청크 데이터 전송
    socket.value.send(JSON.stringify({ type: "stdin", data: data }));
  }
};
</script>

<template>
  <div class="terminal-wrapper flex flex-col h-screen bg-black">
    <!-- 터미널 출력 영역 -->
    <div ref="terminalContainer" class="flex-1 overflow-hidden" />
    
    <!-- CJK IME 한글 입력 중복 방지 컨트롤러 주입 -->
    <ImeChunkInputHelper 
      :terminalInstance="term"
      @sendStdin="handleSendStdin"
    />
  </div>
</template>
```

---

## 4. 이 패턴의 기대 효과 🚀

* **한글 중복 원천 차단**: 낱자 파편화나 쪼개짐, 이중 복사 현상이 완벽히(100%) 소멸합니다.
* **모바일 및 태블릿 웹 터미널 지원**: 모바일 웹브라우저에서 터미널을 제어할 때 가상 키보드가 영문/한글 상관없이 극도의 안정성으로 키를 인젝션합니다.
* **특수 제어 단축키 제공**: 모바일 환경에서 입력하기 불가능에 가까운 `TAB`, `ESC`, `CTRL+C` 등을 터치 그리드 패널로 쉽게 제공하여 원격 서버 제어가 쾌적해집니다.
* 
// 💡 ARCHITECT NOTE: 
// 이 패널의 버튼들은 단순한 고정식 제어키(Ctrl+C, Enter)에 국한되지 않습니다.
// 본질은 백엔드 PTY 스트림을 파괴하지 않고 안전하게 데이터를 밀어 넣는 '청크 창구'입니다.
// 
// [활용 팁] 이 구조를 커스터마이즈하여 자주 쓰는 복잡한 인프라 스크립트, Git 연쇄 명령어, 
// 혹은 AI 프롬프트 주입(Injection) 매크로 버튼 등으로 자유롭게 확장하여 사용하세요.
// 어떤 긴 문장이든 단 하나의 원자적(Atomic) 스트림으로 깨짐 없이 안전하게 발송됩니다.
