<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const BOARD_SIZE = 10
const SHIP_SIZES = [5, 4, 3, 3, 2]
const WS_URL = 'wss://socketsbay.com/wss/v2/1/demo/'

type GameState = 'LOBBY' | 'MATCHMAKING' | 'PLACING' | 'WAITING_OPPONENT' | 'PLAYING' | 'GAME_OVER'

const state = ref<GameState>('LOBBY')
const roomId = ref('')
const inputRoomId = ref('')
const isMyTurn = ref(false)
const socket = ref<WebSocket | null>(null)
const connectionStatus = ref('Đang lặp kết nối...')
const winner = ref<'me' | 'opponent' | null>(null)

// 0: empty, 1: ship, 2: miss, 3: hit
const myBoard = ref<number[][]>(createEmptyBoard())
const oppBoard = ref<number[][]>(createEmptyBoard())

const myHp = ref(17) // 5+4+3+3+2 = 17
const oppHp = ref(17)

// Placing phase
const currentShipIndex = ref(0)
const isHorizontal = ref(true)

function createEmptyBoard() {
  return Array(BOARD_SIZE)
    .fill(0)
    .map(() => Array(BOARD_SIZE).fill(0))
}

function connectWs() {
  connectionStatus.value = 'Đang kết nối WebSocket...'
  socket.value = new WebSocket(WS_URL)

  socket.value.onopen = () => {
    connectionStatus.value = 'Đã kết nối máy chủ'
  }

  socket.value.onmessage = (event) => {
    try {
      const data = JSON.parse(event.data)
      if (data.game !== 'battleship-vibe') return

      // Matchmaking
      if (
        state.value === 'MATCHMAKING' &&
        data.type === 'MATCH_REQUEST' &&
        data.sender !== socket.value
      ) {
        // Find a random player
        roomId.value = data.roomId
        state.value = 'PLACING'
        sendWs({ type: 'MATCH_ACCEPTED', roomId: roomId.value })
      } else if (
        state.value === 'MATCHMAKING' &&
        data.type === 'MATCH_ACCEPTED' &&
        data.roomId === roomId.value
      ) {
        state.value = 'PLACING'
      }

      if (data.roomId !== roomId.value) return // Ignore other rooms

      if (data.type === 'JOIN_ROOM') {
        if (state.value === 'LOBBY') {
          roomId.value = data.roomId
          state.value = 'PLACING'
        }
      }

      if (data.type === 'READY') {
        if (state.value === 'WAITING_OPPONENT') {
          state.value = 'PLAYING'
          isMyTurn.value = false // Host goes second, joiner goes first
        }
      }

      if (state.value === 'PLAYING') {
        if (data.type === 'ATTACK') {
          handleReceiveAttack(data.r, data.c)
        } else if (data.type === 'ATTACK_RESULT') {
          handleAttackResult(data.r, data.c, data.hit)
        } else if (data.type === 'GAME_OVER') {
          winner.value = 'me'
          state.value = 'GAME_OVER'
        }
      }
    } catch {
      // Not JSON
    }
  }

  socket.value.onclose = () => {
    connectionStatus.value = 'Mất kết nối'
  }
}

function sendWs(data: Record<string, unknown>) {
  if (socket.value && socket.value.readyState === WebSocket.OPEN) {
    socket.value.send(JSON.stringify({ ...data, game: 'battleship-vibe', roomId: roomId.value }))
  }
}

function createRoom() {
  roomId.value = Math.random().toString(36).substring(2, 8).toUpperCase()
  state.value = 'PLACING'
  isMyTurn.value = true // Host goes first
}

function joinRoom() {
  if (!inputRoomId.value) return
  roomId.value = inputRoomId.value.toUpperCase()
  state.value = 'PLACING'
  isMyTurn.value = false
  sendWs({ type: 'JOIN_ROOM', roomId: roomId.value })
}

function randomMatch() {
  state.value = 'MATCHMAKING'
  roomId.value = Math.random().toString(36).substring(2, 8).toUpperCase()
  sendWs({ type: 'MATCH_REQUEST', roomId: roomId.value })
}

function placeShip(r: number, c: number) {
  if (state.value !== 'PLACING') return
  const size = SHIP_SIZES[currentShipIndex.value]
  if (!size) return

  // Check bounds
  if (isHorizontal.value && c + size > BOARD_SIZE) return
  if (!isHorizontal.value && r + size > BOARD_SIZE) return

  // Check overlap
  for (let i = 0; i < size; i++) {
    const nr = isHorizontal.value ? r : r + i
    const nc = isHorizontal.value ? c + i : c
    if (myBoard.value[nr][nc] !== 0) return
  }

  // Place
  for (let i = 0; i < size; i++) {
    const nr = isHorizontal.value ? r : r + i
    const nc = isHorizontal.value ? c + i : c
    myBoard.value[nr][nc] = 1
  }

  currentShipIndex.value++
  if (currentShipIndex.value >= SHIP_SIZES.length) {
    state.value = 'WAITING_OPPONENT'
    sendWs({ type: 'READY' })
  }
}

function attack(r: number, c: number) {
  if (state.value !== 'PLAYING' || !isMyTurn.value || oppBoard.value[r][c] !== 0) return
  sendWs({ type: 'ATTACK', r, c })
  isMyTurn.value = false
}

function handleReceiveAttack(r: number, c: number) {
  const isHit = myBoard.value[r][c] === 1
  if (isHit) {
    myBoard.value[r][c] = 3 // hit
    myHp.value--
    if (myHp.value <= 0) {
      sendWs({ type: 'GAME_OVER' })
      winner.value = 'opponent'
      state.value = 'GAME_OVER'
    }
  } else {
    myBoard.value[r][c] = 2 // miss
  }
  sendWs({ type: 'ATTACK_RESULT', r, c, hit: isHit })
  isMyTurn.value = true // Now it's my turn
}

function handleAttackResult(r: number, c: number, hit: boolean) {
  if (hit) {
    oppBoard.value[r][c] = 3
    oppHp.value--
  } else {
    oppBoard.value[r][c] = 2
  }
}

onMounted(() => {
  connectWs()
})

onUnmounted(() => {
  if (socket.value) socket.value.close()
})

const getCellClass = (val: number, isOpponent = false) => {
  if (val === 0) return 'bg-blue-300 hover:bg-blue-400'
  if (val === 1) return isOpponent ? 'bg-blue-300' : 'bg-gray-700'
  if (val === 2) return 'bg-white' // miss
  if (val === 3) return 'bg-red-500' // hit
  return 'bg-blue-300'
}
</script>

<template>
  <div
    class="min-h-screen bg-slate-900 text-slate-100 flex flex-col items-center py-6 px-4 font-sans sm:px-0"
  >
    <!-- Header -->
    <div class="w-full max-w-md flex justify-between items-center mb-6">
      <router-link to="/" class="text-blue-400 hover:text-blue-300 font-medium"
        >← Trang chủ</router-link
      >
      <div class="text-xs text-gray-400 text-right">
        <div>{{ connectionStatus }}</div>
        <div v-if="roomId">
          Phòng: <span class="font-bold text-white">{{ roomId }}</span>
        </div>
      </div>
    </div>

    <!-- 1. Lobby -->
    <div
      v-if="state === 'LOBBY' || state === 'MATCHMAKING'"
      class="w-full max-w-md bg-slate-800 p-6 rounded-2xl shadow-xl flex flex-col gap-4"
    >
      <h1 class="text-3xl font-black text-center text-blue-400 mb-2">BATTLESHIP</h1>
      <p class="text-sm text-gray-400 text-center mb-4">Tác giả: nughnguyen</p>

      <div
        v-if="state === 'MATCHMAKING'"
        class="text-center py-10 animate-pulse text-yellow-400 font-bold"
      >
        Đang tìm đối thủ ngẫu nhiên...
      </div>

      <template v-else>
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
            @keyup.enter="joinRoom"
          />
          <button
            @click="joinRoom"
            class="px-6 bg-slate-700 hover:bg-slate-600 border border-slate-600 rounded-xl font-bold transition-all"
          >
            VÀO
          </button>
        </div>

        <button
          @click="randomMatch"
          class="w-full py-3 mt-4 bg-purple-600 hover:bg-purple-500 rounded-xl font-bold text-white transition-all shadow-lg shadow-purple-500/30 flex justify-center items-center gap-2"
        >
          GHÉP NGẪU NHIÊN 🎲
        </button>
      </template>
    </div>

    <!-- 2. Placing Ships -->
    <div
      v-if="state === 'PLACING'"
      class="w-full max-w-md bg-slate-800 p-6 rounded-2xl shadow-xl flex flex-col items-center"
    >
      <h2 class="text-xl font-bold mb-4 text-yellow-400">Đặt Hạm Đội Của Bạn</h2>
      <div class="mb-4 flex justify-between w-full items-center text-sm">
        <div>
          Tàu tiếp theo:
          <span class="font-bold text-white">{{ SHIP_SIZES[currentShipIndex] }} ô</span>
        </div>
        <button
          @click="isHorizontal = !isHorizontal"
          class="px-4 py-2 bg-slate-700 rounded-lg shadow whitespace-nowrap"
        >
          Xoay: <span class="font-bold text-blue-400">{{ isHorizontal ? 'Ngang' : 'Dọc' }}</span>
        </button>
      </div>
      <!-- Grid -->
      <div
        class="grid grid-cols-10 gap-1 sm:gap-1.5 w-full aspect-square bg-slate-700 p-2 rounded-xl"
      >
        <template v-for="(row, rIndex) in myBoard" :key="'p' + rIndex">
          <div
            v-for="(cell, cIndex) in row"
            :key="'p' + rIndex + '-' + cIndex"
            class="w-full h-full rounded-sm sm:rounded cursor-pointer transition-colors"
            :class="getCellClass(cell)"
            @click="placeShip(rIndex, cIndex)"
          ></div>
        </template>
      </div>
    </div>

    <!-- 3. Waiting -->
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
        bạn bè!
      </p>
    </div>

    <!-- 4. Battle -->
    <div v-if="state === 'PLAYING'" class="w-full max-w-md flex flex-col gap-6">
      <!-- Status -->
      <div
        class="bg-slate-800 p-4 rounded-xl flex justify-between items-center shadow-lg border"
        :class="isMyTurn ? 'border-green-500' : 'border-red-500'"
      >
        <div class="font-bold text-lg" :class="isMyTurn ? 'text-green-400' : 'text-red-400'">
          {{ isMyTurn ? 'ĐẾN LƯỢT BẠN' : 'ĐỐI THỦ ĐANG BẮN...' }}
        </div>
        <div class="text-sm font-bold text-slate-400">
          Máu: <span class="text-red-400">{{ myHp }}/17</span>
        </div>
      </div>

      <!-- Opponent Board (Attack Target) -->
      <div class="bg-slate-800 p-3 rounded-2xl shadow-xl">
        <p class="text-center font-bold text-slate-400 mb-2">LÃNH THỔ ĐỊCH</p>
        <div
          class="grid grid-cols-10 gap-1 w-full aspect-square bg-slate-700 p-1.5 rounded-xl"
          :class="{ 'opacity-60 cursor-not-allowed': !isMyTurn }"
        >
          <template v-for="(row, rIndex) in oppBoard" :key="'o' + rIndex">
            <div
              v-for="(cell, cIndex) in row"
              :key="'o' + rIndex + '-' + cIndex"
              class="w-full h-full rounded-sm transition-colors"
              :class="[
                getCellClass(cell, true),
                isMyTurn ? 'cursor-pointer hover:bg-blue-200' : '',
              ]"
              @click="attack(rIndex, cIndex)"
            ></div>
          </template>
        </div>
      </div>

      <!-- My Board -->
      <div class="bg-slate-800 p-3 rounded-2xl shadow-xl opacity-80 scale-95 mx-4">
        <p class="text-center font-bold text-slate-400 mb-2 text-sm">LÃNH THỔ CỦA BẠN</p>
        <div class="grid grid-cols-10 gap-0.5 w-full aspect-square bg-slate-700 p-1 rounded-xl">
          <template v-for="(row, rIndex) in myBoard" :key="'m' + rIndex">
            <div
              v-for="(cell, cIndex) in row"
              :key="'m' + rIndex + '-' + cIndex"
              class="w-full h-full rounded-sm"
              :class="getCellClass(cell)"
            ></div>
          </template>
        </div>
      </div>
    </div>

    <!-- 5. Game Over -->
    <div
      v-if="state === 'GAME_OVER'"
      class="w-full max-w-md bg-slate-800 p-8 rounded-2xl shadow-xl text-center transform transition-all"
    >
      <div class="text-6xl mb-4">{{ winner === 'me' ? '🏆' : '💥' }}</div>
      <h2
        class="text-3xl font-black mb-2"
        :class="winner === 'me' ? 'text-green-400' : 'text-red-500'"
      >
        {{ winner === 'me' ? 'CHIẾN THẮNG!' : 'BẠN ĐÃ THUA!' }}
      </h2>
      <button
        @click="() => window.location.reload()"
        class="mt-6 px-8 py-3 bg-blue-600 hover:bg-blue-500 rounded-xl font-bold text-white transition-all"
      >
        CHƠI LẠI
      </button>
    </div>
  </div>
</template>
