<script setup lang="ts">
import { ref, onUnmounted, nextTick } from 'vue'

const WORDS = [
  'Mèo',
  'Chó',
  'Con vịt',
  'Bãi biển',
  'Máy bay',
  'Ngôi nhà',
  'Bầu trời',
  'Mặt trời',
  'Cái ghế',
  'Tivi',
  'Máy tính',
  'Bóng đá',
  'Chiếc xe',
  'Hoa hồng',
  'Quả táo',
  'Cái chai',
  'Bàn phím',
  'Cái bát',
  'Đồng hồ',
  'Bút chì',
  'Bác sĩ',
  'Giáo viên',
  'Cây cối',
  'Sư tử',
  'Con thỏ',
  'Quả dưa hấu',
  'Đám mây',
  'Chiếc ô',
  'Kính lúp',
  'Tên lửa',
  'Cái chổi',
  'Răng',
]

type GameState = 'LOBBY' | 'WAITING_OPPONENT' | 'PLAYING' | 'END_ROUND'

const state = ref<GameState>('LOBBY')
const roomId = ref('')
const inputRoomId = ref('')
const connectionStatus = ref('Chưa kết nối')

// PeerJS Networking
declare let Peer: unknown
const peer = ref<unknown>(null)
const conn = ref<unknown>(null)

// Role logic
const myRole = ref<'drawer' | 'guesser' | null>(null)
const currentWord = ref('')
const score = ref({ me: 0, opponent: 0 })
const chatLog = ref<{ sender: string; text: string; isCorrect?: boolean }[]>([])
const currentGuess = ref('')
const roundWinnerMessage = ref('')

// Canvas Drawing
const canvasRef = ref<HTMLCanvasElement | null>(null)
const isDrawing = ref(false)
const ctx = ref<CanvasRenderingContext2D | null>(null)
const selectedColor = ref('#000000')
const selectedSize = ref(3)

let lastX = 0
let lastY = 0

// Inject PeerJS script dynamically
function initPeerJs(): Promise<boolean> {
  return new Promise((resolve) => {
    if (window.hasOwnProperty('Peer')) return resolve(true)
    const script = document.createElement('script')
    script.src = 'https://unpkg.com/peerjs@1.5.2/dist/peerjs.min.js'
    script.onload = () => resolve(true)
    document.head.appendChild(script)
  })
}

async function createRoom() {
  await initPeerJs()
  connectionStatus.value = 'Đang tạo phòng P2P...'
  roomId.value = Math.random().toString(36).substring(2, 8).toUpperCase()

  peer.value = new Peer('vibescribble-' + roomId.value)

  peer.value.on('open', () => {
    connectionStatus.value = 'Phòng đã sẵn sàng'
    state.value = 'WAITING_OPPONENT'
    myRole.value = 'drawer'
    currentWord.value = getRandomWord()
  })

  peer.value.on('connection', (c: unknown) => {
    // Someone joined
    conn.value = c
    setupConn()
    connectionStatus.value = 'Đã kết nối với đối thủ!'

    // Give connection a moment to establish, then start
    setTimeout(() => {
      conn.value.send(JSON.stringify({ type: 'SYNC_START', word: currentWord.value }))
      startGameAs('drawer')
    }, 1000)
  })

  peer.value.on('error', (err: unknown) => {
    connectionStatus.value = 'Lỗi tạo phòng: ' + (err as { type: string }).type
    state.value = 'LOBBY'
  })
}

async function joinRoomBtn() {
  if (!inputRoomId.value) return
  await initPeerJs()
  connectionStatus.value = 'Đang tìm phòng...'
  roomId.value = inputRoomId.value.toUpperCase()

  peer.value = new Peer()

  peer.value.on('open', () => {
    // Attempt to connect to the host
    conn.value = peer.value.connect('vibescribble-' + roomId.value)

    conn.value.on('open', () => {
      connectionStatus.value = 'Đã kết nối!'
      setupConn()
      state.value = 'WAITING_OPPONENT'
      myRole.value = 'guesser'
    })

    conn.value.on('error', () => {
      connectionStatus.value = 'Không tìm thấy phòng'
      state.value = 'LOBBY'
    })
  })

  peer.value.on('error', () => {
    connectionStatus.value = 'Phòng không tồn tại hoặc lỗi mạng'
    state.value = 'LOBBY'
  })
}

function setupConn() {
  conn.value.on('data', (dataStr: string) => {
    try {
      const data = JSON.parse(dataStr)

      if (data.type === 'SYNC_START') {
        if (state.value === 'WAITING_OPPONENT' || state.value === 'END_ROUND') {
          currentWord.value = data.word
          startGameAs('guesser')
        }
      }

      if (state.value === 'PLAYING') {
        if (data.type === 'DRAW') {
          drawLine(data.x0, data.y0, data.x1, data.y1, data.color, data.size, false)
        } else if (data.type === 'CLEAR') {
          clearCanvas(false)
        } else if (data.type === 'GUESS') {
          chatLog.value.push({ sender: 'Đối thủ', text: data.text })
        } else if (data.type === 'GUESS_CORRECT') {
          chatLog.value.push({ sender: 'Hệ thống', text: 'Đối thủ đã đoán ĐÚNG!', isCorrect: true })
          endRound('opponent')
        } else if (data.type === 'NEW_ROUND') {
          currentWord.value = data.word
          startGameAs('guesser') // Opponent is the new drawer
        }
      }
    } catch {
      // Ignore bad payload
    }
  })

  conn.value.on('close', () => {
    connectionStatus.value = 'Đối thủ đã ngắt kết nối'
    state.value = 'LOBBY'
    alert('Đối thủ đã thoát phòng!')
  })
}

function sendWs(data: Record<string, unknown>) {
  if (conn.value && conn.value.open) {
    conn.value.send(JSON.stringify(data))
  }
}

function getRandomWord() {
  return WORDS[Math.floor(Math.random() * WORDS.length)]
}

function startGameAs(role: 'drawer' | 'guesser') {
  myRole.value = role
  state.value = 'PLAYING'
  chatLog.value = []
  clearCanvas(false)
  nextTick(() => {
    initCanvas()
  })
}

function endRound(winner: 'me' | 'opponent') {
  state.value = 'END_ROUND'
  if (winner === 'me') {
    score.value.me++
    roundWinnerMessage.value = 'Bạn ĐÃ ĐÚNG! Từ khóa là: ' + currentWord.value
  } else {
    score.value.opponent++
    roundWinnerMessage.value = 'Đối thủ đã đoán đúng! Từ khóa: ' + currentWord.value
  }

  if (winner === 'me') {
    // Guesser won, they become the Drawer next
    setTimeout(() => {
      currentWord.value = getRandomWord()
      startGameAs('drawer')
      sendWs({ type: 'NEW_ROUND', word: currentWord.value })
    }, 4000)
  }
}

function sendGuess() {
  if (!currentGuess.value.trim() || myRole.value !== 'guesser') return

  const g = currentGuess.value.trim()
  chatLog.value.push({ sender: 'Bạn', text: g })

  if (g.toLowerCase() === currentWord.value.toLowerCase()) {
    chatLog.value.push({ sender: 'Hệ thống', text: 'Bạn đã đoán ĐÚNG!', isCorrect: true })
    sendWs({ type: 'GUESS_CORRECT' })
    endRound('me')
  } else {
    sendWs({ type: 'GUESS', text: g })
  }
  currentGuess.value = ''
}

// Canvas
function initCanvas() {
  if (!canvasRef.value) return
  ctx.value = canvasRef.value.getContext('2d') as CanvasRenderingContext2D
  resizeCanvas()
  window.addEventListener('resize', resizeCanvas)
}

function resizeCanvas() {
  if (canvasRef.value) {
    const parent = canvasRef.value.parentElement
    if (parent) {
      canvasRef.value.width = parent.clientWidth
      canvasRef.value.height = parent.clientWidth // Square
    }
  }
}

function startDrawing(e: MouseEvent | TouchEvent) {
  if (myRole.value !== 'drawer' || state.value !== 'PLAYING') return
  isDrawing.value = true
  const pos = getMousePos(e)
  lastX = pos.x
  lastY = pos.y
}

function draw(e: MouseEvent | TouchEvent) {
  if (!isDrawing.value || myRole.value !== 'drawer' || state.value !== 'PLAYING') return
  e.preventDefault()
  const pos = getMousePos(e)
  drawLine(lastX, lastY, pos.x, pos.y, selectedColor.value, selectedSize.value, true)
  lastX = pos.x
  lastY = pos.y
}

function stopDrawing() {
  if (myRole.value !== 'drawer' || state.value !== 'PLAYING') return
  isDrawing.value = false
}

function drawLine(
  x0: number,
  y0: number,
  x1: number,
  y1: number,
  color: string,
  size: number,
  emit: boolean,
) {
  if (!ctx.value) return
  ctx.value.beginPath()
  ctx.value.moveTo(x0, y0)
  ctx.value.lineTo(x1, y1)
  ctx.value.strokeStyle = color
  ctx.value.lineWidth = size
  ctx.value.lineCap = 'round'
  ctx.value.stroke()
  ctx.value.closePath()

  if (emit) {
    sendWs({ type: 'DRAW', x0, y0, x1, y1, color, size })
  }
}

function getMousePos(e: MouseEvent | TouchEvent) {
  if (!canvasRef.value) return { x: 0, y: 0 }
  const rect = canvasRef.value.getBoundingClientRect()
  const clientX = 'touches' in e ? e.touches[0].clientX : (e as MouseEvent).clientX
  const clientY = 'touches' in e ? e.touches[0].clientY : (e as MouseEvent).clientY
  return {
    x: clientX - rect.left,
    y: clientY - rect.top,
  }
}

function clearCanvasBtn() {
  if (myRole.value !== 'drawer') return
  clearCanvas(true)
}

function clearCanvas(emit: boolean) {
  if (ctx.value && canvasRef.value) {
    ctx.value.clearRect(0, 0, canvasRef.value.width, canvasRef.value.height)
  }
  if (emit) {
    sendWs({ type: 'CLEAR' })
  }
}

onUnmounted(() => {
  if (conn.value) conn.value.close()
  if (peer.value) peer.value.destroy()
  window.removeEventListener('resize', resizeCanvas)
})
</script>

<template>
  <div
    class="min-h-screen bg-slate-900 text-slate-100 flex flex-col items-center py-6 px-4 font-sans sm:px-0"
  >
    <!-- Header -->
    <div class="w-full max-w-xl flex justify-between items-center mb-4">
      <router-link to="/" class="text-blue-400 hover:text-blue-300 font-medium whitespace-nowrap"
        >← Trang chủ</router-link
      >
      <div class="text-xs text-slate-400 text-right">
        <div>{{ connectionStatus }}</div>
        <div v-if="roomId">
          Phòng: <span class="font-bold text-white">{{ roomId }}</span>
        </div>
      </div>
    </div>

    <!-- 1. Lobby -->
    <div
      v-if="state === 'LOBBY'"
      class="w-full max-w-md bg-slate-800 p-6 rounded-2xl shadow-xl flex flex-col gap-4"
    >
      <h1 class="text-3xl font-black text-center text-blue-400 mb-2">SCRIBBLE</h1>
      <p class="text-sm text-slate-400 text-center mb-4">Đoán Chữ Vẽ Hình - Tác giả: nughnguyen</p>

      <button
        @click="createRoom"
        class="w-full py-3 bg-blue-600 hover:bg-blue-500 rounded-xl font-bold text-white transition-all shadow-lg shadow-blue-500/30"
      >
        TẠO PHÒNG MỚI
      </button>

      <div class="relative flex items-center py-2">
        <div class="grow border-t border-slate-700"></div>
        <span class="shrink-0 mx-4 text-slate-500 text-sm">hoặc</span>
        <div class="grow border-t border-slate-700"></div>
      </div>

      <div class="flex gap-2">
        <input
          v-model="inputRoomId"
          placeholder="Nhập ID Phòng..."
          class="flex-1 bg-slate-700 border border-slate-600 rounded-xl px-4 py-3 focus:outline-none focus:ring-2 focus:ring-blue-500 text-white uppercase"
          maxlength="6"
          @keyup.enter="joinRoomBtn"
        />
        <button
          @click="joinRoomBtn"
          class="px-6 bg-slate-700 hover:bg-slate-600 border border-slate-600 rounded-xl font-bold transition-all"
        >
          VÀO
        </button>
      </div>

      <p class="text-xs text-slate-500 text-center mt-2">
        Tính năng ghép ngẫu nhiên đã được vô hiệu hoá để đảm bảo đường truyền P2P tốc độ cao.
      </p>
    </div>

    <!-- 2. Waiting -->
    <div
      v-if="state === 'WAITING_OPPONENT'"
      class="w-full max-w-md bg-slate-800 p-8 rounded-2xl shadow-xl text-center"
    >
      <div
        class="animate-spin w-12 h-12 border-4 border-blue-500 border-t-transparent rounded-full mx-auto mb-4"
      ></div>
      <h2 class="text-xl font-bold text-white mb-2">Đang chờ đối thủ...</h2>
      <p class="text-slate-400">
        Hãy gửi
        <span class="text-yellow-400 font-bold px-1 bg-slate-900 rounded">{{ roomId }}</span> cho
        bạn bè bấm Vào phòng!
      </p>
    </div>

    <!-- 3. Playing/End Round -->
    <div
      v-if="state === 'PLAYING' || state === 'END_ROUND'"
      class="w-full max-w-xl bg-slate-800 p-4 rounded-2xl shadow-xl flex flex-col gap-4"
    >
      <!-- Score & Status -->
      <div
        class="flex justify-between items-center bg-slate-900 p-3 rounded-lg border border-slate-700"
      >
        <div class="text-green-400 font-bold">Bạn: {{ score.me }}</div>

        <!-- Word display -->
        <div class="text-center font-black text-xl tracking-widest text-white">
          <template v-if="myRole === 'drawer' || state === 'END_ROUND'">
            {{ currentWord }}
          </template>
          <template v-else>
            <span class="opacity-50 tracking-widest border-b-2 border-slate-500 pb-1">
              {{ currentWord.replace(/[^\s]/g, '_ ') }}
            </span>
            <div class="text-xs text-slate-400 mt-1 font-normal tracking-normal">
              {{ currentWord.length }} chữ cái
            </div>
          </template>
        </div>

        <div class="text-red-400 font-bold">Địch: {{ score.opponent }}</div>
      </div>

      <div
        v-if="state === 'END_ROUND'"
        class="p-6 text-center text-xl font-bold text-yellow-400 animate-pulse bg-slate-700 rounded-xl"
      >
        {{ roundWinnerMessage }}<br />
        <span class="text-sm text-slate-300 font-normal mt-2 block">Chuẩn bị đổi phiên...</span>
      </div>

      <!-- Canvas Area -->
      <div
        class="relative w-full aspect-square border-2 border-slate-700 rounded-xl overflow-hidden bg-white shadow-inner"
        style="touch-action: none"
      >
        <canvas
          ref="canvasRef"
          class="w-full h-full cursor-crosshair"
          @mousedown="startDrawing"
          @mousemove="draw"
          @mouseup="stopDrawing"
          @mouseout="stopDrawing"
          @touchstart.prevent="startDrawing"
          @touchmove.prevent="draw"
          @touchend.prevent="stopDrawing"
        ></canvas>

        <div v-if="myRole === 'guesser'" class="absolute inset-0 pointer-events-none"></div>
      </div>

      <!-- Drawer Tools -->
      <div
        v-if="myRole === 'drawer'"
        class="flex justify-between items-center bg-slate-700 p-2 rounded-lg"
      >
        <div class="flex gap-2">
          <!-- Colors -->
          <button
            v-for="c in ['#000000', '#EF4444', '#3B82F6', '#10B981', '#F59E0B']"
            :key="c"
            class="w-8 h-8 rounded-full border-2 transition-transform"
            :class="selectedColor === c ? 'border-white scale-110' : 'border-slate-500 opacity-80'"
            :style="{ backgroundColor: c }"
            @click="selectedColor = c"
          ></button>
        </div>
        <div class="flex gap-2 items-center">
          <input type="range" v-model.number="selectedSize" min="1" max="15" class="w-24" />
          <button
            @click="clearCanvasBtn"
            class="bg-red-600 hover:bg-red-500 px-3 py-1 rounded text-white text-sm font-bold shadow"
          >
            XOÁ
          </button>
        </div>
      </div>

      <!-- Guesser Input & Chat -->
      <div class="flex flex-col gap-2 bg-slate-900 border border-slate-700 p-3 rounded-xl h-40">
        <div
          class="flex-1 overflow-y-auto space-y-1 text-sm font-medium flex flex-col-reverse custom-scrollbar"
        >
          <div
            v-for="(msg, i) in [...chatLog].reverse()"
            :key="i"
            :class="
              msg.isCorrect
                ? 'text-green-400 font-bold bg-green-900/30 p-1 rounded'
                : 'text-slate-300'
            "
          >
            <span :class="msg.sender === 'Bạn' ? 'text-blue-400' : 'text-slate-500'"
              >{{ msg.sender }}:</span
            >
            {{ msg.text }}
          </div>
        </div>

        <div class="flex gap-2" v-if="myRole === 'guesser' && state === 'PLAYING'">
          <input
            v-model="currentGuess"
            @keyup.enter="sendGuess"
            placeholder="Nhắn tin để đoán..."
            class="flex-1 bg-slate-800 border border-slate-600 rounded-lg px-3 py-2 text-white text-sm focus:outline-none focus:border-blue-500"
          />
          <button @click="sendGuess" class="bg-blue-600 px-4 py-2 rounded-lg font-bold">Gửi</button>
        </div>
        <div v-else-if="myRole === 'drawer'" class="text-center text-slate-500 text-sm mt-2 italic">
          Đối thủ đang đoán, hãy tiếp tục vẽ để hỗ trợ...
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.custom-scrollbar::-webkit-scrollbar {
  width: 6px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: transparent;
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  background-color: #475569;
  border-radius: 10px;
}
</style>
