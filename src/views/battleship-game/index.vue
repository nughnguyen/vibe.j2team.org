<script setup lang="ts">
import { ref } from 'vue'

const BOARD_SIZE = 10
const maxMisses = 20

// 0: empty, 1: ship part, 2: hit, 3: miss
const board = ref<number[][]>(
  Array(BOARD_SIZE)
    .fill(0)
    .map(() => Array(BOARD_SIZE).fill(0)),
)

const gameState = ref<'idle' | 'playing' | 'won' | 'lost'>('idle')
const misses = ref(0)
const hits = ref(0)
const message = ref('Nhấn Start để bắt đầu!')
const targetHits = ref(0)

const placeShips = () => {
  // Reset board
  for (let i = 0; i < BOARD_SIZE; i++) {
    for (let j = 0; j < BOARD_SIZE; j++) {
      board.value[i][j] = 0
    }
  }

  const shipSizes = [5, 4, 3, 3, 2]
  let totalParts = 0

  shipSizes.forEach((size) => {
    let placed = false
    while (!placed) {
      const isHorizontal = Math.random() > 0.5
      const row = Math.floor(Math.random() * (isHorizontal ? BOARD_SIZE : BOARD_SIZE - size + 1))
      const col = Math.floor(Math.random() * (isHorizontal ? BOARD_SIZE - size + 1 : BOARD_SIZE))

      let canPlace = true
      for (let i = 0; i < size; i++) {
        const r = isHorizontal ? row : row + i
        const c = isHorizontal ? col + i : col
        if (board.value[r][c] !== 0) {
          canPlace = false
          break
        }
      }

      if (canPlace) {
        for (let i = 0; i < size; i++) {
          const r = isHorizontal ? row : row + i
          const c = isHorizontal ? col + i : col
          board.value[r][c] = 1
        }
        placed = true
        totalParts += size
      }
    }
  })

  targetHits.value = totalParts
}

const startGame = () => {
  placeShips()
  gameState.value = 'playing'
  misses.value = 0
  hits.value = 0
  message.value = 'Tìm và tiêu diệt hạm đội địch!'
}

const attack = (row: number, col: number) => {
  if (gameState.value !== 'playing') return

  const cell = board.value[row][col]

  if (cell === 2 || cell === 3) return // Already attacked here

  if (cell === 1) {
    board.value[row][col] = 2 // Hit
    hits.value++
    message.value = 'Trúng mục tiêu! Tiếp tục nào.'
    if (hits.value >= targetHits.value) {
      gameState.value = 'won'
      message.value = 'Chúc mừng! Bạn đã tiêu diệt toàn bộ hạm đội địch.'
    }
  } else if (cell === 0) {
    board.value[row][col] = 3 // Miss
    misses.value++
    message.value = 'Trượt rồi! Địch không ở đó.'
    if (misses.value >= maxMisses) {
      gameState.value = 'lost'
      message.value = 'Hết lượt! Bạn đã thua.'
      revealShips()
    }
  }
}

const revealShips = () => {
  for (let i = 0; i < BOARD_SIZE; i++) {
    for (let j = 0; j < BOARD_SIZE; j++) {
      if (board.value[i][j] === 1) {
        board.value[i][j] = 4 // Revealed unhit ship
      }
    }
  }
}

const getCellClass = (cell: number) => {
  switch (cell) {
    case 0:
    case 1:
      return 'bg-blue-400 hover:bg-blue-500 cursor-pointer'
    case 2:
      return 'bg-red-500 cursor-not-allowed'
    case 3:
      return 'bg-gray-400 cursor-not-allowed'
    case 4:
      return 'bg-green-500 cursor-not-allowed' // Revealed ship at end
    default:
      return 'bg-blue-400'
  }
}
</script>

<template>
  <div class="min-h-screen bg-gray-900 text-white flex flex-col items-center py-10 font-sans">
    <div class="mb-4">
      <router-link to="/" class="text-blue-400 hover:text-blue-300 underline font-medium">
        &larr; Về trang chủ (Home)
      </router-link>
    </div>

    <h1 class="text-4xl font-bold mb-2 text-blue-300 tracking-wider uppercase">BattleShip</h1>
    <p class="text-gray-400 mb-6 text-center max-w-md">
      Tìm và phá hủy 5 tàu địch đang ẩn náu. Bạn chỉ được phép bắn trượt {{ maxMisses }} lần!
    </p>

    <div class="bg-gray-800 p-6 rounded-xl shadow-2xl border border-gray-700">
      <div class="flex justify-between items-center mb-6">
        <div class="flex gap-4 text-lg">
          <div class="px-4 py-2 bg-gray-700 rounded-lg">
            <span class="text-red-400 font-bold">HIT:</span> {{ hits }}/{{ targetHits || '?' }}
          </div>
          <div class="px-4 py-2 bg-gray-700 rounded-lg">
            <span class="text-gray-400 font-bold">MISS:</span> {{ misses }}/{{ maxMisses }}
          </div>
        </div>
        <button
          @click="startGame"
          class="px-6 py-2 bg-blue-600 hover:bg-blue-500 rounded-lg font-bold transition-colors shadow-lg"
        >
          {{ gameState === 'idle' ? 'START GAME' : 'RESTART' }}
        </button>
      </div>

      <div
        class="text-center mb-4 min-h-[1.5rem] font-medium"
        :class="{
          'text-yellow-400': gameState === 'playing',
          'text-green-400 text-xl': gameState === 'won',
          'text-red-400 text-xl': gameState === 'lost',
        }"
      >
        {{ message }}
      </div>

      <div
        class="grid grid-cols-10 gap-1 sm:gap-2 w-full max-w-lg mx-auto bg-blue-900 p-2 rounded-lg"
        :class="{ 'opacity-50 pointer-events-none': gameState !== 'playing' }"
      >
        <template v-for="(row, rIndex) in board" :key="rIndex">
          <div
            v-for="(cell, cIndex) in row"
            :key="`${rIndex}-${cIndex}`"
            class="w-8 h-8 sm:w-10 sm:h-10 rounded transition-colors duration-200"
            :class="getCellClass(cell)"
            @click="attack(rIndex, cIndex)"
          >
            <!-- Hit icon -->
            <div v-if="cell === 2" class="w-full h-full flex items-center justify-center">
              <svg
                xmlns="http://www.w3.org/2000/svg"
                class="h-6 w-6 text-white"
                viewBox="0 0 20 20"
                fill="currentColor"
              >
                <path
                  fill-rule="evenodd"
                  d="M3.172 5.172a4 4 0 015.656 0L10 6.343l1.172-1.171a4 4 0 115.656 5.656L10 17.657l-6.828-6.829a4 4 0 010-5.656z"
                  clip-rule="evenodd"
                />
              </svg>
            </div>
            <!-- Miss circle -->
            <div v-if="cell === 3" class="w-full h-full flex items-center justify-center">
              <div class="w-3 h-3 bg-white rounded-full opacity-50"></div>
            </div>
            <!-- Revealed ship part -->
            <div v-if="cell === 4" class="w-full h-full flex items-center justify-center">
              <svg
                xmlns="http://www.w3.org/2000/svg"
                class="h-6 w-6 text-white"
                fill="none"
                viewBox="0 0 24 24"
                stroke="currentColor"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M13 10V3L4 14h7v7l9-11h-7z"
                />
              </svg>
            </div>
          </div>
        </template>
      </div>

      <!-- Legend -->
      <div class="mt-6 flex justify-center gap-6 text-sm text-gray-300">
        <div class="flex items-center gap-2">
          <div class="w-4 h-4 bg-red-500 rounded"></div>
          Trúng
        </div>
        <div class="flex items-center gap-2">
          <div class="w-4 h-4 bg-gray-400 rounded"></div>
          Trượt
        </div>
        <div class="flex items-center gap-2">
          <div class="w-4 h-4 bg-green-500 rounded"></div>
          Tàu địch (Khi thua)
        </div>
      </div>
    </div>
  </div>
</template>
