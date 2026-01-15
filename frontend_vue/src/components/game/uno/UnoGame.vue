<template>
  <div class="uno-game" v-if="isGameActive">
    <button @click="closeGame" class="close-btn">离开</button>

    <!-- AI Card Area -->
    <div class="ai-card-area">
      <!-- Avatar Small Box -->
      <div class="player-info">
        <div class="avatar-box" v-if="avatarImageUrl">
          <div class="avatar-img" :style="{ backgroundImage: `url(${avatarImageUrl})` }"></div>
        </div>
        <div class="player-name">{{ gameStore.character }}</div>
        <div class="card-count">{{ aiHand.length }} 张卡牌</div>
      </div>
      <div class="ai-cards">
        <div
          v-for="(card, index) in aiHand"
          :key="`ai-${index}`"
          class="card card-back"
          :style="{ transform: `translateX(${index * 20}px)` }"
        >
          <img src="/pictures/games/uno/card_back.png" alt="卡背" />
        </div>
      </div>
    </div>

    <!-- Center Play Area -->
    <div class="center-area">
      <GameDialog />
      <!-- Draw Pile -->
      <div class="draw-pile" @click="handleDrawCard">
        <div class="pile-label">抽牌堆 ({{ drawPile.length }})</div>
        <div class="card-stack">
          <img src="/pictures/games/uno/card_back.png" alt="抽牌堆" />
        </div>
      </div>

      <!-- Discard Pile -->
      <div class="discard-pile">
        <div class="pile-label">弃牌堆</div>
        <div class="card-stack" v-if="discardPile.length > 0">
          <img
            :src="getCardImage(discardPile[discardPile.length - 1]!)"
            :alt="discardPile[discardPile.length - 1]!.display"
          />
        </div>
      </div>

      <!-- Game Status -->
      <div class="game-status">
        <div class="current-turn">{{ gameStatus }}</div>
        <div
          v-if="currentColor"
          class="current-color"
          :style="{ color: getColorHex(currentColor) }"
        >
          当前颜色: {{ getColorText(currentColor) }}
        </div>
        <button v-if="selectedCardIndex !== null" @click="playSelectedCard" class="play-card-btn">
          出牌
        </button>
      </div>
    </div>

    <!-- Player Card Area -->
    <div class="player-card-area">
      <div class="player-info">
        <div class="player-name">莱姆</div>
        <div class="player-count">{{ playerHand.length }} 张卡牌</div>
      </div>
      <div class="player-cards">
        <div
          v-for="(card, index) in playerHand"
          :key="`player-${index}`"
          class="card"
          :class="{ playable: isCardPlayable(card), selected: selectedCardIndex === index }"
          @click="selectCard(index)"
          :style="{ transform: `translateX(${index * 50}px)` }"
        >
          <img :src="getCardImage(card)" :alt="card.display" />
        </div>
      </div>
    </div>

    <!-- Color Picker Modal -->
    <div v-if="showColorPicker" class="color-picker-modal">
      <div class="color-picker-content">
        <h3>选择颜色</h3>
        <div class="color-options">
          <div
            v-for="color in ['red', 'blue', 'green', 'yellow']"
            :key="color"
            @click="selectWildColor(color)"
            class="color-option"
            :style="{ backgroundColor: getColorHex(color) }"
          >
            {{ getColorText(color) }}
          </div>
        </div>
      </div>
    </div>

    <!-- Game Message -->
    <transition name="fade">
      <div v-if="gameMessage" class="game-message">
        {{ gameMessage }}
      </div>
    </transition>

    <!-- Loading Overlay -->
    <div v-if="waitingForGameStart" class="loading-overlay">
      <div class="loading-content">
        <div class="loading-spinner"></div>
        <div class="loading-text">正在初始化游戏...</div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, computed, onMounted, onUnmounted } from 'vue'
import { scriptHandler } from '@/api/websocket/handlers/script-handler'
import { useGameStore } from '@/stores/modules/game'
import { useUIStore } from '@/stores/modules/ui/ui'
import { EMOTION_CONFIG, EMOTION_CONFIG_EMO } from '@/controllers/emotion/config'
import GameDialog from '../standard/GameDialog.vue'

// Card Types
interface Card {
  color: 'red' | 'blue' | 'green' | 'yellow' | 'wild'
  value: string // '0'-'9', 'skip', 'stop', 'draw_two', 'wild', 'wild_draw_four'
  display: string
}

// Game State
const isGameActive = ref(false)
const playerHand = ref<Card[]>([])
const aiHand = ref<Card[]>([])
const drawPile = ref<Card[]>([])
const discardPile = ref<Card[]>([])
const currentColor = ref<string>('')
const gameStatus = ref('点击"开始游戏"开始')
const selectedCardIndex = ref<number | null>(null)
const showColorPicker = ref(false)
const gameMessage = ref('')
const isPlayerTurn = ref(true)
const pendingWildCard = ref<Card | null>(null)
const avatarImageUrl = ref('')
const waitingForAIResponse = ref(false)
const waitingForGameStart = ref(false)
const aiPlayableCards = ref<Array<{ card: Card; index: number }>>([])

const gameStore = useGameStore()
const uiStore = useUIStore()

// Get avatar URL
const updateAvatarImage = () => {
  const character = gameStore.character
  const clothes_name = gameStore.avatar.clothes_name ?? 'default'
  const emotion = gameStore.avatar.emotion
  const emotionConfig = EMOTION_CONFIG[emotion] || EMOTION_CONFIG['正常']

  if (emotion === 'AI思考') {
    avatarImageUrl.value = ''
    return
  }

  const avatarUrl = `${emotionConfig?.avatar}/${clothes_name}`
  if (!gameStore.script.isRunning) {
    avatarImageUrl.value = emotionConfig?.avatar ? avatarUrl : ''
    return
  }

  avatarImageUrl.value = `/api/v1/chat/character/get_script_avatar/${character}/${clothes_name}/${EMOTION_CONFIG_EMO[emotion]}`
}

// Watch for avatar changes
watch(
  () => [gameStore.avatar.emotion, gameStore.avatar.clothes_name, gameStore.character],
  () => {
    updateAvatarImage()
  },
  { immediate: true },
)

// Initialize deck
const initializeDeck = (): Card[] => {
  const deck: Card[] = []
  const colors: ('red' | 'blue' | 'green' | 'yellow')[] = ['red', 'blue', 'green', 'yellow']

  // Number cards (0-9)
  colors.forEach((color) => {
    deck.push({ color, value: '0', display: `${color}_0` })
    for (let i = 1; i <= 9; i++) {
      deck.push({ color, value: String(i), display: `${color}_${i}` })
      deck.push({ color, value: String(i), display: `${color}_${i}` })
    }
  })

  // Action cards
  colors.forEach((color) => {
    for (let i = 0; i < 2; i++) {
      deck.push({ color, value: 'skip', display: `${color}_skip` })
      deck.push({ color, value: 'stop', display: `${color}_stop` })
      deck.push({ color, value: 'draw_two', display: `${color}_draw_two` })
    }
  })

  // Wild cards
  for (let i = 0; i < 4; i++) {
    deck.push({ color: 'wild', value: 'wild', display: 'wild' })
    deck.push({ color: 'wild', value: 'wild_draw_four', display: 'wild_draw_four' })
  }

  return shuffle(deck)
}

// Shuffle function
const shuffle = (array: Card[]): Card[] => {
  const newArray = [...array]
  for (let i = newArray.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[newArray[i]!, newArray[j]!] = [newArray[j]!, newArray[i]!]
  }
  return newArray
}

// Get card image path
const getCardImage = (card: Card): string => {
  return `/pictures/games/uno/${card.display}.png`
}

// Get color hex
const getColorHex = (color: string): string => {
  const colorMap: Record<string, string> = {
    red: '#E74C3C',
    blue: '#3498DB',
    green: '#2ECC71',
    yellow: '#F1C40F',
    wild: '#34495E',
  }
  return colorMap[color] || '#FFFFFF'
}

// Get color text
const getColorText = (color: string): string => {
  const textMap: Record<string, string> = {
    red: '红色',
    blue: '蓝色',
    green: '绿色',
    yellow: '黄色',
    wild: '万能',
  }
  return textMap[color] || color
}

// Check if card is playable
const isCardPlayable = (card: Card): boolean => {
  if (!isPlayerTurn.value) return false

  const topCard = discardPile.value[discardPile.value.length - 1]
  if (!topCard) return true

  // Wild cards can always be played
  if (card.color === 'wild') return true

  // Same color or same value
  return card.color === currentColor.value || card.value === topCard.value
}

// Start game
const startGame = () => {
  const deck = initializeDeck()
  drawPile.value = deck
  discardPile.value = []
  playerHand.value = []
  aiHand.value = []

  // Deal 7 cards to each player
  for (let i = 0; i < 7; i++) {
    const playerCard = drawPile.value.pop()
    const aiCard = drawPile.value.pop()
    if (playerCard) playerHand.value.push(playerCard)
    if (aiCard) aiHand.value.push(aiCard)
  }

  // First card to discard pile
  let firstCard = drawPile.value.pop()
  while (firstCard && firstCard.color === 'wild') {
    drawPile.value.unshift(firstCard)
    firstCard = drawPile.value.pop()
  }

  if (!firstCard) return

  discardPile.value.push(firstCard)
  currentColor.value = firstCard.color

  isGameActive.value = true
  isPlayerTurn.value = true
  gameStatus.value = '你的回合'

  // Send game start message to LLM and wait for response
  const gameInfo = `
    你现在要玩UNO游戏了，规则如下:
    1.发牌 & 起始:每人发7张牌,翻开一张作为弃牌堆的起始牌。
    2.出牌：跟上家的牌，颜色、数字或功能一致即可；无牌可出或不想出，需摸牌，可立即打出，也可结束回合。
    3.赢家：最先打完所有牌的玩家贏。
    UNO游戏开始!你有${aiHand.value.length}张牌。当前弃牌堆顶牌是：${firstCard.display}（${getColorText(firstCard.color)}）。
    第一回合你不需要出牌，只需要骂玩家是杂鱼即可。
    游戏中的回答限制在一行文字，不要输出多余的回答。
  `
  waitingForGameStart.value = true
  document.getElementById('sendButton')?.click()
  scriptHandler.sendMessage(gameInfo)
}

// Select card
const selectCard = (index: number) => {
  if (!isPlayerTurn.value) {
    showMessage('现在不是你的回合！')
    return
  }

  const card = playerHand.value[index]
  if (!card || !isCardPlayable(card)) {
    showMessage('这张牌无法出！')
    return
  }

  selectedCardIndex.value = selectedCardIndex.value === index ? null : index
}

// Play selected card
const playSelectedCard = () => {
  if (selectedCardIndex.value === null) return

  const card = playerHand.value[selectedCardIndex.value]
  if (!card) return

  // Handle wild cards
  if (card.color === 'wild') {
    pendingWildCard.value = card
    showColorPicker.value = true
    return
  }

  playCard(card, selectedCardIndex.value)
}

// Play card
const playCard = (card: Card, index: number) => {
  document.getElementById('sendButton')?.click()
  playerHand.value.splice(index, 1)
  discardPile.value.push(card)
  // Only set currentColor for non-wild cards
  // For wild cards, the color is already set in selectWildColor
  if (card.color !== 'wild') {
    currentColor.value = card.color
  }
  selectedCardIndex.value = null

  // Handle special cards
  handleCardEffect(card, 'player')

  // Check win
  if (playerHand.value.length === 0) {
    gameStatus.value = '你赢了！'
    showMessage('恭喜你赢得了游戏！')
    setTimeout(() => closeGame(), 3000)
    return
  }

  // AI turn
  if (card.value !== 'skip' && card.value !== 'stop') {
    isPlayerTurn.value = false
    gameStatus.value = `${gameStore.character}的回合`

    // Send to LLM
    // const message = `玩家出了一张牌：${card.display}（${getColorText(card.color)} ${card.value}）。现在轮到你了，你的手牌数量：${aiHand.value.length}。当前颜色是${getColorText(currentColor.value)}。请选择出牌或抽牌。`
    // scriptHandler.sendMessage(message)

    // AI plays after delay
    setTimeout(() => aiPlay(), 2000)
  }
}

// Select wild color
const selectWildColor = (color: string) => {
  showColorPicker.value = false
  const cardIndex = selectedCardIndex.value
  if (pendingWildCard.value && cardIndex !== null) {
    const card = pendingWildCard.value
    // Set the chosen color BEFORE playing the card
    currentColor.value = color
    playCard(card, cardIndex)
    pendingWildCard.value = null
  }
}

// Handle card effect
const handleCardEffect = (card: Card, player: string) => {
  switch (card.value) {
    case 'draw_two':
      if (player === 'player') {
        drawCards(aiHand.value, 2)
        showMessage(`${gameStore.character}抽了2张牌！`)
      } else {
        drawCards(playerHand.value, 2)
        showMessage('你抽了2张牌！')
      }
      break
    case 'wild_draw_four':
      if (player === 'player') {
        drawCards(aiHand.value, 4)
        showMessage(`${gameStore.character}抽了4张牌！`)
      } else {
        drawCards(playerHand.value, 4)
        showMessage('你抽了4张牌！')
      }
      break
    case 'skip':
      if (player !== 'player') {
        document.getElementById('sendButton')?.click()
      }
      showMessage(`跳过${player === 'player' ? `${gameStore.character}` : '玩家'}的回合！`)
      break
    case 'stop':
      if (player !== 'player') {
        document.getElementById('sendButton')?.click()
      }
      showMessage(`跳过${player === 'player' ? `${gameStore.character}` : '玩家'}的回合！`)
      break
  }
}

// Draw cards
const drawCards = (hand: Card[], count: number) => {
  for (let i = 0; i < count; i++) {
    if (drawPile.value.length === 0) {
      reshuffleDeck()
    }
    if (drawPile.value.length > 0) {
      const card = drawPile.value.pop()
      if (card) hand.push(card)
    }
  }
}

// Handle draw card
const handleDrawCard = () => {
  if (!isPlayerTurn.value) {
    showMessage('现在不是你的回合！')
    return
  }

  document.getElementById('sendButton')?.click()

  if (drawPile.value.length === 0) {
    reshuffleDeck()
  }

  if (drawPile.value.length > 0) {
    const card = drawPile.value.pop()
    if (card) {
      playerHand.value.push(card)
      showMessage('你抽了一张牌')

      // End turn
      isPlayerTurn.value = false
      gameStatus.value = `${gameStore.character}的回合`
      setTimeout(() => aiPlay(), 2000)
    }
  }
}

// Reshuffle deck
const reshuffleDeck = () => {
  if (discardPile.value.length <= 1) return

  const topCard = discardPile.value.pop()
  if (!topCard) return

  drawPile.value = shuffle([...discardPile.value])
  discardPile.value = [topCard]
  showMessage('抽牌堆已重新洗牌！')
}

// Parse AI response to extract card choice
const parseAIResponse = (response: string) => {
  if (!waitingForAIResponse.value) return

  const text = response.toLowerCase().trim()
  console.log('解析AI响应:', text, '可用牌数:', aiPlayableCards.value.length)

  // Check if AI wants to draw
  if (text.includes('抽牌') || text.includes('摸牌')) {
    waitingForAIResponse.value = false
    executeAIDraw()
    return
  }

  // Try to extract card index (第N张牌)
  const indexMatch = text.match(/第?\s*(\d+)\s*张/)
  if (indexMatch && indexMatch[1]) {
    const cardNum = parseInt(indexMatch[1])
    if (cardNum >= 1 && cardNum <= aiPlayableCards.value.length) {
      waitingForAIResponse.value = false
      executeAIPlayCard(cardNum - 1) // Convert to 0-based index
      return
    }
  }

  // Try to match card name (e.g., red_5, blue_skip)
  for (let i = 0; i < aiPlayableCards.value.length; i++) {
    const { card } = aiPlayableCards.value[i]!
    const cardName = card.display.toLowerCase()
    if (text.includes(cardName)) {
      waitingForAIResponse.value = false
      executeAIPlayCard(i)
      return
    }
  }

  // If no valid match, use random
  console.log('无法解析AI响应，使用随机选择')
  waitingForAIResponse.value = false
  const randomIndex = Math.floor(Math.random() * aiPlayableCards.value.length)
  executeAIPlayCard(randomIndex)
}

// Execute AI draw card action
const executeAIDraw = () => {
  if (drawPile.value.length === 0) {
    reshuffleDeck()
  }
  if (drawPile.value.length > 0) {
    const card = drawPile.value.pop()
    if (card) {
      aiHand.value.push(card)
      showMessage(`${gameStore.character}抽了一张牌`)
      // Too many sending will cause game slow
      // const drawResultMessage = `${gameStore.character}抽了一张牌，现在有${aiHand.value.length}张牌。`
      // scriptHandler.sendMessage(drawResultMessage)
    }
  }

  // Player's turn
  if (aiHand.value.length > 0) {
    isPlayerTurn.value = true
    gameStatus.value = '你的回合'
  }
}

// Execute AI play card action
const executeAIPlayCard = (playableIndex: number) => {
  const selectedCard = aiPlayableCards.value[playableIndex]
  if (!selectedCard) {
    // Fallback to random
    const randomIndex = Math.floor(Math.random() * aiPlayableCards.value.length)
    const fallbackCard = aiPlayableCards.value[randomIndex]
    if (!fallbackCard) return
    executeAIPlayCard(randomIndex)
    return
  }

  const { card, index } = selectedCard

  aiHand.value.splice(index, 1)
  discardPile.value.push(card)

  // Handle wild cards
  if (card.color === 'wild') {
    const colors: ('red' | 'blue' | 'green' | 'yellow')[] = ['red', 'blue', 'green', 'yellow']
    const selectedColor = colors[Math.floor(Math.random() * colors.length)]
    if (selectedColor) {
      currentColor.value = selectedColor
      showMessage(`${gameStore.character}出了${card.display}，选择了${getColorText(selectedColor)}`)
    }
  } else {
    currentColor.value = card.color
    showMessage(`${gameStore.character}出了${card.display}`)
  }

  // Handle special cards
  handleCardEffect(card, 'ai')

  // Check AI win
  if (aiHand.value.length === 0) {
    gameStatus.value = `${gameStore.character}赢了`
    showMessage(`${gameStore.character}赢得了游戏！`)
    setTimeout(() => closeGame(), 5000)
    return
  }

  // Send AI move result to LLM
  // const resultMessage = `${gameStore.character}出了：${card.display}（${getColorText(card.color)} ${card.value}）。${gameStore.character}还剩${aiHand.value.length}张牌。`

  // scriptHandler.sendMessage(resultMessage)

  // Check if AI should continue playing (skip/stop cards)
  if (card.value === 'skip' || card.value === 'stop') {
    // AI continues to play
    gameStatus.value = `${gameStore.character}的回合`
    setTimeout(() => aiPlay(), 1500)
  } else {
    // Player's turn
    if (aiHand.value.length > 0) {
      isPlayerTurn.value = true
      gameStatus.value = '你的回合'
    }
  }
}

// AI play logic
const aiPlay = () => {
  const topCard = discardPile.value[discardPile.value.length - 1]
  if (!topCard) return

  // Find playable cards
  const playableCards = aiHand.value
    .map((card, index) => ({ card, index }))
    .filter(({ card }) => {
      return (
        card.color === 'wild' || card.color === currentColor.value || card.value === topCard.value
      )
    })

  // Store playable cards for AI response parsing
  aiPlayableCards.value = playableCards

  if (playableCards.length > 0) {
    // Build card options info for AI
    const cardOptionsInfo = playableCards
      .map(({ card }, idx) => {
        return `${idx + 1}. ${card.display} (${getColorText(card.color)} ${card.value})`
      })
      .join(', ')

    // Send card options to AI and wait for response
    const optionsMessage = `你可以出以下${playableCards.length}张牌：${cardOptionsInfo}。当前颜色是${getColorText(currentColor.value)}。请告诉我你要出哪张牌（例如："我出第1张牌"或"我出red_5"），不要返回多余信息。`

    waitingForAIResponse.value = true
    scriptHandler.sendMessage(optionsMessage)

    // Set timeout fallback in case AI doesn't respond
    setTimeout(() => {
      if (waitingForAIResponse.value) {
        console.log('AI响应超时，使用随机选择')
        waitingForAIResponse.value = false
        const randomIndex = Math.floor(Math.random() * aiPlayableCards.value.length)
        executeAIPlayCard(randomIndex)
      }
    }, 60000) // 60 second timeout
  } else {
    // No playable cards - must draw
    // const drawMessage = `你没有可以出的牌，当前颜色是${getColorText(currentColor.value)}。你需要抽一张牌。`
    // scriptHandler.sendMessage(drawMessage)

    setTimeout(() => {
      executeAIDraw()
    }, 1500)
  }
}

const showMessage = (message: string) => {
  gameMessage.value = message
  setTimeout(() => {
    gameMessage.value = ''
  }, 2000)
}

const closeGame = () => {
  isGameActive.value = false
  playerHand.value = []
  aiHand.value = []
  drawPile.value = []
  discardPile.value = []
  selectedCardIndex.value = null
  waitingForAIResponse.value = false
  waitingForGameStart.value = false
}

console.log('uistore', uiStore)

// watch UI message => parse AI action or game start
watch(
  () => uiStore.showCharacterLine,
  (newLine) => {
    console.log('ai reply:', newLine)
    if (waitingForGameStart.value && newLine && newLine.trim()) {
      console.log('收到游戏开始AI响应:', newLine)
      waitingForGameStart.value = false
      showMessage('游戏开始！')
    } else if (waitingForAIResponse.value && newLine && newLine.trim()) {
      console.log('收到AI响应:', newLine)
      parseAIResponse(newLine)
    }
  },
)

defineExpose({
  startGame,
})
</script>

<style scoped>
@reference "tailwindcss";

.uno-game {
  @apply fixed w-screen h-screen backdrop-blur-[30px] z-[9999] flex flex-col overflow-y-scroll overflow-x-hidden left-0 top-0;
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(30px) saturate(150%);
  -webkit-backdrop-filter: blur(30px) saturate(150%);
}

.uno-game::before {
  @apply content-[''] absolute pointer-events-none animate-[liquidMove_20s_ease-in-out_infinite] inset-0;
  background:
    radial-gradient(circle at 10% 20%, rgba(255, 255, 255, 0.08) 0%, transparent 30%),
    radial-gradient(circle at 90% 30%, rgba(255, 255, 255, 0.05) 0%, transparent 40%),
    radial-gradient(circle at 50% 80%, rgba(255, 255, 255, 0.06) 0%, transparent 50%);
  animation: liquidMove 20s ease-in-out infinite;
}

@keyframes liquidMove {
  0%,
  100% {
    opacity: 0.5;
    transform: scale(1) rotate(0deg);
  }
  50% {
    opacity: 0.8;
    transform: scale(1.1) rotate(5deg);
  }
}

.game-header {
  @apply flex justify-between items-center backdrop-blur-[20px] shadow-[0_4px_15px_rgba(0,0,0,0.1)] px-10 py-5 border-b-[rgba(255,255,255,0.1)] border-b border-solid;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(20px) saturate(180%);
  -webkit-backdrop-filter: blur(20px) saturate(180%);
}

.game-title {
  @apply text-[32px] font-[bold] text-[white];
  text-shadow: 0 0 20px rgba(255, 255, 255, 0.5);
}
.close-btn {
  @apply absolute text-[white] w-[100px] h-10 rounded text-2xl cursor-pointer transition-all duration-[0.3s] border-2 border-solid border-[rgba(255,255,255,0.3)] right-0 hover:scale-110;
  background: rgba(255, 255, 255, 0.2);
}
.close-btn:hover {
  background: rgba(255, 255, 255, 0.3);
}

/* Avatar Box */
.avatar-box {
  @apply w-[100px] h-[100px] overflow-hidden shadow-[0_8px_25px_rgba(0,0,0,0.3),inset_0_2px_10px_rgba(255,255,255,0.2)] backdrop-blur-[10px] transition-all duration-[0.3s] ease-[ease] mb-5 rounded-[50%] border-4 border-solid border-[rgba(255,255,255,0.3)] hover:shadow-[0_12px_35px_rgba(0,0,0,0.4),inset_0_2px_10px_rgba(255,255,255,0.3)] hover:border-[rgba(255,255,255,0.5)] hover:scale-110;
  background: rgba(255, 255, 255, 0.05);
}
.avatar-img {
  @apply w-full h-full bg-cover bg-center bg-no-repeat transition-transform duration-[0.3s] ease-[ease];
}
.avatar-box:hover .avatar-img {
  @apply scale-[1.15];
}

/* AI Card Area */
.ai-card-area {
  @apply min-h-[200px] flex flex-row justify-center gap-[100px] items-center p-5;
  background: rgba(255, 255, 255, 0.02);
}
.player-info {
  @apply flex gap-5 text-[white] text-lg items-center mb-[15px];
}
.player-name {
  @apply font-[bold];
  text-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
}
.card-count,
.player-count {
  @apply text-[#ffd700];
}
.ai-cards {
  @apply flex relative h-[150px];
}
/* Center Area */
.center-area {
  @apply flex-1 flex justify-center items-center gap-[60px] p-10;
}
.draw-pile,
.discard-pile {
  @apply flex flex-col items-center gap-[15px];
}
.pile-label {
  @apply text-[white] text-lg font-[bold];
  text-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
}
.card-stack {
  @apply w-[120px] h-[180px] shadow-[0_10px_30px_rgba(0,0,0,0.5)] transition-transform duration-[0.3s] cursor-pointer rounded-xl;
}
.draw-pile .card-stack:hover {
  @apply -translate-y-2.5 scale-105;
}
.card-stack img {
  @apply w-full h-full rounded-xl;
}
.game-status {
  @apply flex flex-col items-center gap-2.5 border shadow-[0_8px_32px_rgba(0,0,0,0.2),inset_0_1px_2px_rgba(255,255,255,0.1)] px-[30px] py-5 rounded-[20px] border-solid border-[rgba(255,255,255,0.15)];
  background: rgba(255, 255, 255, 0.08);
  backdrop-filter: saturate(180%);
  -webkit-backdrop-filter: saturate(180%);
}
.current-turn {
  @apply text-[white] text-2xl font-[bold];
  text-shadow: 0 0 15px rgba(255, 255, 255, 0.7);
}
.current-color {
  @apply text-xl font-[bold];
  text-shadow: 0 0 10px currentColor;
}
/* Player Card Area */
.player-card-area {
  @apply min-h-[250px] flex flex-col items-center backdrop-blur-[15px] shadow-[0_-4px_15px_rgba(0,0,0,0.1)] p-5 border-t-[rgba(255,255,255,0.1)] border-t border-solid;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: saturate(150%);
  -webkit-backdrop-filter: blur(15px) saturate(150%);
}
.player-cards {
  @apply flex relative h-[180px] mb-5;
}
.card {
  @apply absolute w-[100px] h-[150px] shadow-[0_5px_15px_rgba(0,0,0,0.3)] transition-all duration-[0.3s] ease-[ease] cursor-pointer z-[1] rounded-[10px];
}
.card img {
  @apply w-full h-full pointer-events-none rounded-[10px];
}
.card.playable {
  @apply cursor-pointer hover:-translate-y-5 hover:shadow-[0_15px_40px_rgba(255,255,255,0.4)] hover:z-[100];
}
.card.selected {
  @apply -translate-y-5 shadow-[0_0_40px_rgba(255,215,0,0.8)] z-[101];
}
.card-back {
  @apply cursor-default;
}
.play-card-btn {
  @apply text-[white] text-xl font-[bold] cursor-pointer shadow-[0_5px_15px_rgba(0,0,0,0.3)] transition-all duration-[0.3s] px-10 py-[15px] rounded-[25px] border-[none] hover:translate-y-[-3px] hover:shadow-[0_8px_20px_rgba(0,0,0,0.4)];
  background: linear-gradient(135deg, #8998d8 0%, #f5f6ff 100%);
}

/* Color Picker Modal */
.color-picker-modal {
  @apply fixed w-full h-full flex justify-center items-center z-[10000] backdrop-blur-[5px] left-0 top-0;
  background: rgba(0, 0, 0, 0.7);
}
.color-picker-content {
  @apply shadow-[0_10px_50px_rgba(0,0,0,0.5)] p-10 rounded-[20px];
  background: white;
}
.color-picker-content h3 {
  @apply text-[28px] text-center text-[#333] mt-0 mb-[30px] mx-0;
}
.color-options {
  @apply grid grid-cols-[repeat(2,1fr)] gap-5;
}
.color-option {
  @apply text-xl font-[bold] text-center text-[white] cursor-pointer transition-all duration-[0.3s] shadow-[0_5px_15px_rgba(0,0,0,0.3)] p-[30px] rounded-[15px] hover:shadow-[0_8px_25px_rgba(0,0,0,0.4)] hover:scale-110;
}

/* Game Message */
.game-message {
  @apply fixed -translate-x-2/4 -translate-y-2/4 text-[white] text-2xl font-[bold] z-[10001] backdrop-blur-[10px] shadow-[0_10px_40px_rgba(0,0,0,0.5)] px-[50px] py-[30px] rounded-[15px] left-2/4 top-2/4;
  background: rgba(0, 0, 0, 0.8);
}
.fade-enter-active,
.fade-leave-active {
  @apply transition-opacity duration-[0.3s];
}
.fade-enter-from,
.fade-enter-from,
.fade-leave-to {
  @apply opacity-0;
}

/* Loading Overlay */
.loading-overlay {
  @apply fixed w-full h-full flex justify-center items-center z-[10002] backdrop-blur-[5px] left-0 top-0;
  background: rgba(0, 0, 0, 0.8);
}
.loading-content {
  @apply flex flex-col items-center gap-5;
}
.loading-spinner {
  @apply w-[60px] h-[60px] rounded-[50%] border-t-white border-4 border-solid border-[rgba(255,255,255,0.3)];
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

.loading-text {
  @apply text-[white] text-2xl font-[bold];
  text-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
}
</style>
