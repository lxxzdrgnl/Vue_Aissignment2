<template>
  <div class="app-root">
    <!-- Layer 1: Fixed Background Elements -->
    <div class="background-layer" aria-hidden="true">
      <div class="bg-gradient"></div>
      <div class="cloud cloud--left"></div>
      <div class="cloud cloud--right"></div>

      <div
          v-for="s in stars"
          :key="s.id"
          class="star"
          :style="starStyle(s)"
      ></div>

      <div class="moon-wrap">
        <svg viewBox="0 0 200 200" class="moon" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="보름달">
          <defs>
            <radialGradient id="gMoon" cx="35%" cy="30%">
              <stop offset="0%" stop-color="#fff9d9" />
              <stop offset="100%" stop-color="#f0c85a" />
            </radialGradient>
            <filter id="softGlow" x="-50%" y="-50%" width="200%" height="200%">
              <feGaussianBlur stdDeviation="6" result="b" />
              <feMerge>
                <feMergeNode in="b" />
                <feMergeNode in="SourceGraphic" />
              </feMerge>
            </filter>
          </defs>
          <g filter="url(#softGlow)">
            <circle cx="100" cy="100" r="86" fill="url(#gMoon)" />
            <circle cx="70" cy="80" r="8" fill="#f7e0a2" opacity="0.8" />
            <circle cx="120" cy="115" r="6" fill="#f7e0a2" opacity="0.7" />
            <circle cx="92" cy="58" r="4" fill="#f7e0a2" opacity="0.6" />
          </g>
        </svg>
        <img src="/tokkie.png" alt="Rabbit on the moon" class="rabbit-icon" />
      </div>

      <div
          v-for="lan in lanterns"
          :key="lan.id"
          class="lantern"
          :style="lanternStyle(lan)"
      >
        <svg viewBox="0 0 60 100" width="60" height="100" xmlns="http://www.w3.org/2000/svg">
          <defs>
            <linearGradient id="lanGrad" x1="0" x2="1">
              <stop offset="0%" stop-color="#ffd89b" />
              <stop offset="100%" stop-color="#ff7a7a" />
            </linearGradient>
          </defs>
          <g>
            <ellipse cx="30" cy="45" rx="24" ry="32" fill="url(#lanGrad)" stroke="#8b3b2a" stroke-width="1.5" />
            <rect x="26" y="72" width="8" height="18" rx="3" fill="#6b3b2a" />
            <path d="M8 40 Q30 20 52 40" stroke="#8b3b2a" stroke-width="1" fill="none" opacity="0.6" />
            <g class="flame" transform="translate(30,58)">
              <path d="M0 0 C4 -6, 8 -6, 2 -18 C-2 -10 -6 -6 0 0" fill="#fff1b8" />
              <path d="M0 -3 C2 -7, 6 -7, 1 -14 C-1 -10 -3 -7 0 -3" fill="#ffd166" />
            </g>
          </g>
        </svg>
      </div>
    </div>

    <!-- Layer 2: Scrollable Foreground Content -->
    <div class="foreground-layer">
      <main class="content" role="main">
        <h1 class="title">풍성한 한가위 되세요</h1>
        <p class="subtitle">달빛 아래 토끼가 뛰노는 따뜻한 밤. 소중한 분들과 함께하는 풍성한 한가위 보내세요.</p>

        <section class="comment-panel" aria-label="추석 코멘트">
          <label for="comment" class="visually-hidden">추석 인사 입력</label>
          <div class="input-row">
            <input
                id="comment"
                v-model="newComment"
                @keydown.enter.prevent="addComment"
                type="text"
                maxlength="200"
                placeholder="추석 인사를 남겨보세요 (Enter로 제출)"
                class="text-input"
                aria-label="추석 인사 입력"
            />
            <button class="btn" @click="addComment" aria-label="코멘트 남기기">남기기</button>
          </div>

          <div class="controls">
            <span class="count">댓글 수: {{ comments.length }}</span>
            <button class="btn--ghost" @click="clearComments" v-if="comments.length">모두 삭제</button>
          </div>

          <ul class="comment-list">
            <li v-for="c in comments" :key="c.id" class="comment-item">
              <div class="comment-text">{{ c.text }}</div>
              <div class="comment-meta">
                <time :dateTime="c.createdAt">{{ formatDate(c.createdAt) }}</time>
                <button class="del" @click="deleteComment(c.id)" aria-label="댓글 삭제">삭제</button>
              </div>
            </li>
          </ul>
        </section>

        <footer class="attribution">(샘플) Vue + TS + Vite로 만든 한가위 데모 페이지</footer>
      </main>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'

// ----------------------------
// 타입 선언
// ----------------------------
type Star = { id: number; x: number; y: number; size: number; delay: number; duration: number }
type Lantern = { id: number; left: number; duration: number; sway: number; delay: number }
type Comment = { id: number; text: string; createdAt: string }

// ----------------------------
// 설정값들 (원하시면 조정 가능)
// ----------------------------
const STAR_COUNT = 72
const LANTERN_SPAWN_MS = 1800 // 등불 생성 간격(밀리초)
const MAX_LANTERNS = 8
const LOCAL_KEY = 'chuseok_comments_v1'

// ----------------------------
// 반응형 상태
// ----------------------------
const stars = ref<Star[]>([])
const lanterns = ref<Lantern[]>([])
const comments = ref<Comment[]>([])
const newComment = ref('')

// Interval id를 저장해서 컴포넌트 해제 시 정리
let spawnInterval: number | null = null

// ----------------------------
// 유틸: 랜덤 정수/실수
// ----------------------------
const randInt = (a: number, b: number) => Math.floor(Math.random() * (b - a + 1)) + a
const rand = (a: number, b: number) => Math.random() * (b - a) + a

// ----------------------------
// 스타 생성: 위치/크기/깜빡임 타이밍을 미리 계산
// ----------------------------
function generateStars(n = STAR_COUNT) {
  const arr: Star[] = []
  for (let i = 0; i < n; i++) {
    arr.push({
      id: Date.now() + i + Math.floor(Math.random() * 9999),
      x: rand(1, 99),
      y: rand(2, 98), // Y-range increased for full screen
      size: randInt(1, 4),
      delay: Number((rand(0, 4)).toFixed(2)),
      duration: Number((rand(2, 5)).toFixed(2)),
    })
  }
  stars.value = arr
}

// ----------------------------
// 등불 생성/제거 로직
// ----------------------------
function spawnLantern() {
  if (lanterns.value.length >= MAX_LANTERNS) return
  const id = Date.now() + Math.floor(Math.random() * 1000)
  const left = rand(8, 92)
  const duration = rand(10, 18) // 떠오르는 시간 (초)
  const sway = rand(2, 5)
  const delay = 0

  const lan: Lantern = { id, left, duration, sway, delay }
  lanterns.value.push(lan)

  // 애니메이션 끝나면 배열에서 제거 (정리)
  const ttl = (duration + 1) * 1000
  setTimeout(() => {
    lanterns.value = lanterns.value.filter(l => l.id !== id)
  }, ttl)
}

// 스타/등불 초기화 및 주기 세팅
onMounted(() => {
  generateStars(STAR_COUNT)

  // 초반에 몇 개 띄워놓음
  for (let i = 0; i < 3; i++) setTimeout(spawnLantern, i * 600)

  spawnInterval = window.setInterval(spawnLantern, LANTERN_SPAWN_MS)

  // 로컬스토리지에서 코멘트 불러오기
  try {
    const raw = localStorage.getItem(LOCAL_KEY)
    if (raw) comments.value = JSON.parse(raw)
  } catch (e) {
    // ignore
  }
})

onBeforeUnmount(() => {
  if (spawnInterval) clearInterval(spawnInterval)
})

// ----------------------------
// 코멘트 관련 함수: 추가/삭제/포맷
// ----------------------------
function saveComments() {
  try {
    localStorage.setItem(LOCAL_KEY, JSON.stringify(comments.value))
  } catch (e) {
    // 저장 불가 시 무시
  }
}

function addComment() {
  const text = newComment.value.trim()
  if (!text) return
  if (text.length > 200) {
    // 간단한 guard: 너무 길면 200자로 제한
    alert('코멘트는 200자 이내로 입력해주세요.')
    return
  }
  const c: Comment = { id: Date.now(), text, createdAt: new Date().toISOString() }
  comments.value.unshift(c)
  newComment.value = ''
  saveComments()
}

function deleteComment(id: number) {
  comments.value = comments.value.filter(c => c.id !== id)
  saveComments()
}

function clearComments() {
  if (!confirm('모든 댓글을 삭제하시겠습니까?')) return
  comments.value = []
  saveComments()
}

function formatDate(iso: string) {
  try {
    const d = new Date(iso)
    return d.toLocaleString('ko-KR', { year: 'numeric', month: 'short', day: 'numeric', hour: '2-digit', minute: '2-digit' })
  } catch (e) {
    return iso
  }
}

// ----------------------------
// 템플릿에서 쓰이는 스타일 정보 반환
// ----------------------------
function starStyle(s: Star) {
  return {
    left: `${s.x}%`,
    top: `${s.y}%`,
    width: `${s.size}px`,
    height: `${s.size}px`,
    animationDelay: `${s.delay}s`,
    animationDuration: `${s.duration}s`,
    opacity: `${0.6 + Math.random() * 0.4}`,
  }
}

function lanternStyle(l: Lantern) {
  return {
    left: `${l.left}%`,
    animationDuration: `${l.duration}s`,
    ['--sway' as any]: `${l.sway}px`,
  }
}
</script>

<style>
/* Global Styles */
body {
  background-color: #16222A; /* Dark blue fallback */
  color: #E0E0E0;
}
:root {
  /* New blue gradient */
  --bg-top: #16222A;
  --bg-mid: #3A6073;
  --bg-bottom: #16222A;

  --accent: #F7971E; /* A warmer, more vibrant orange/yellow */
  --panel-bg: rgba(22, 34, 42, 0.75); /* Darker, less transparent panel */
  --panel-border: rgba(255, 255, 255, 0.1);
  --text-primary: #F2F2F2;
  --text-secondary: #BDBDBD;
}
</style>

<style scoped>
.app-root {
  font-family: 'Pretendard', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Noto Sans KR', 'Apple SD Gothic Neo', 'Malgun Gothic', sans-serif;
  color: var(--text-primary);
}

/* Layer 1: Fixed background */
.background-layer {
  position: fixed;
  inset: 0;
  z-index: 0;
  overflow: hidden;
}

.bg-gradient {
  position: absolute;
  inset: 0;
  background: linear-gradient(180deg, var(--bg-top), var(--bg-mid) 60%, var(--bg-bottom));
}

.cloud {
  position: absolute;
  width: 30%;
  min-width: 350px;
  height: 150px;
  top: 10%;
  background: linear-gradient(90deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));
  filter: blur(22px);
  border-radius: 50%;
  animation: cloudMove 40s linear infinite alternate;
}
.cloud--left { left: -15%; }
.cloud--right { right: -10%; top: 20%; animation-duration: 55s; }
@keyframes cloudMove { from { transform: translateX(-8%) } to { transform: translateX(8%) } }

.star {
  position: absolute;
  background: #fff;
  border-radius: 50%;
  box-shadow: 0 0 8px #fff, 0 0 14px #fff;
  animation-name: twinkle;
  animation-iteration-count: infinite;
  animation-direction: alternate;
  animation-timing-function: ease-in-out;
}
@keyframes twinkle {
  0% { opacity: 0.2; transform: scale(0.8); }
  100% { opacity: 0.8; transform: scale(1.2); }
}

.moon-wrap {
  position: absolute;
  left: 50%;
  top: 8%;
  transform: translateX(-50%);
  width: 260px;
  height: 260px;
  z-index: 5;
  pointer-events: none;
  animation: moonDrift 15s ease-in-out infinite alternate;
}
@keyframes moonDrift { from { transform: translateX(-50%) translateY(0) } to { transform: translateX(-48%) translateY(-15px) } }

.moon {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
}

.rabbit-icon {
  position: absolute;
  top: 20px;
  right: 20px;
  width: 160px; /* Blatantly bigger */
  height: auto;
  filter: drop-shadow(0 0 8px rgba(0,0,0,0.6));
  animation: rabbitHop 2.5s cubic-bezier(0.42, 0, 0.58, 1) infinite;
}
@keyframes rabbitHop {
  0% { transform: translateY(0px); opacity: 0.9; }
  50% { transform: translateY(-8px) scale(1.05); opacity: 1; }
  100% { transform: translateY(0px); opacity: 0.9; }
}

.lantern {
  position: absolute;
  bottom: -120px;
  z-index: 3;
  transform: translateX(-50%);
  animation-name: lanternFloat;
  animation-timing-function: ease-in;
  animation-iteration-count: 1;
  pointer-events: none;
}
@keyframes lanternFloat {
  from { transform: translateX(-50%) translateY(0) rotate(-2deg) scale(0.9); opacity: 1; }
  to { transform: translateX(calc(-50% + var(--sway, 0px))) translateY(-120vh) rotate(15deg) scale(1.0); opacity: 0; }
}

.flame { animation: flameFlicker 1.2s ease-in-out infinite alternate; transform-origin: center bottom; }
@keyframes flameFlicker { from { transform: scale(1) } to { transform: scale(0.9) } }

/* Layer 2: Foreground */
.foreground-layer {
  position: relative;
  z-index: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start; /* Changed to move content down */
  min-height: 100vh;
  padding: 2rem;
  box-sizing: border-box;
}

.content {
  width: 100%;
  max-width: 800px;
  padding: 32px;
  text-align: center;
  background: var(--panel-bg);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  border: 1px solid var(--panel-border);
  box-shadow: 0 25px 50px -12px rgba(0,0,0,0.5);
  margin-top: 20vh; /* Pushed down from top */
}

.title {
  font-size: 38px;
  font-weight: 700;
  margin: 0;
  color: var(--accent);
  text-shadow: 0 4px 25px rgba(0,0,0,0.5);
}
.subtitle {
  margin-top: 12px;
  color: var(--text-primary);
  font-size: 18px;
  font-weight: 400;
  opacity: 0.9;
}

.comment-panel { margin-top: 32px; text-align: left; }
.input-row { display: flex; gap: 12px; align-items: center; }

.text-input {
  flex: 1;
  padding: 12px 16px;
  border-radius: 12px;
  border: 1px solid var(--panel-border);
  background: rgba(0,0,0,0.2);
  color: var(--text-primary);
  font-size: 16px;
  transition: all 0.2s ease;
}
.text-input::placeholder { color: var(--text-secondary); opacity: 0.7; }
.text-input:focus {
  outline: none;
  border-color: var(--accent);
  box-shadow: 0 0 0 3px rgba(247, 151, 30, 0.2);
}

.btn {
  padding: 12px 20px;
  border-radius: 12px;
  border: none;
  background: var(--accent);
  color: #111;
  font-weight: 700;
  font-size: 16px;
  cursor: pointer;
  transition: transform 0.2s ease, background-color 0.2s ease;
}
.btn:hover { transform: translateY(-2px); }
.btn:active { transform: translateY(0); }

.controls { margin-top: 12px; display: flex; justify-content: space-between; align-items: center; font-size: 14px; }
.count { color: var(--text-secondary); }
.btn--ghost {
  background: transparent;
  color: var(--text-secondary);
  border: 1px solid transparent;
  padding: 6px 10px;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
}
.btn--ghost:hover { color: #ff8a8a; background: rgba(255, 100, 100, 0.1); }

.comment-list { margin-top: 16px; display: grid; gap: 10px; padding: 0; }
.comment-item {
  padding: 14px 16px;
  border-radius: 12px;
  background: rgba(0,0,0,0.15);
  border: 1px solid var(--panel-border);
  display: flex;
  justify-content: space-between;
  align-items: center;
  transition: background-color 0.2s ease;
}
.comment-item:hover { background: rgba(0,0,0,0.2); }

.comment-text { flex: 1; font-weight: 500; }
.comment-meta { display: flex; gap: 12px; align-items: center; font-size: 13px; color: var(--text-secondary); }
.del { background: transparent; color: #aaa; border: none; cursor: pointer; transition: color 0.2s ease; }
.del:hover { color: #ff8a8a; }

.attribution { margin-top: 24px; color: var(--text-secondary); font-size: 13px; opacity: 0.6; }

.visually-hidden { position: absolute !important; height: 1px; width: 1px; overflow: hidden; clip: rect(1px, 1px, 1px, 1px); white-space: nowrap; }

@media (max-width: 600px) {
  .foreground-layer { padding: 1rem; justify-content: flex-start; }
  .moon-wrap { width: 180px; height: 180px; top: 5%; }
  .rabbit-icon {
    width: 120px;
    top: 10px;
    right: 10px;
  }
  .content { margin-top: 25vh; padding: 24px; }
  .title { font-size: 28px; }
  .subtitle { font-size: 16px; }
  .input-row { flex-direction: column; gap: 12px; }
  .text-input, .btn { width: 100%; }
  .comment-item {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }
  .comment-meta {
    width: 100%;
    display: flex;
    justify-content: space-between;
    font-size: 12px;
  }
}
</style>
