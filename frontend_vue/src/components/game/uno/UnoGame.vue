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
      <GameDialog class="uno-game-dialog" />
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
  if (card.value !== 'skip') {
    isPlayerTurn.value = false
    gameStatus.value = 'AI的回合'

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
        showMessage('AI抽了2张牌！')
      } else {
        drawCards(playerHand.value, 2)
        showMessage('你抽了2张牌！')
      }
      break
    case 'wild_draw_four':
      if (player === 'player') {
        drawCards(aiHand.value, 4)
        showMessage('AI抽了4张牌！')
      } else {
        drawCards(playerHand.value, 4)
        showMessage('你抽了4张牌！')
      }
      break
    case 'skip':
      showMessage(`跳过${player === 'player' ? 'AI' : '玩家'}的回合！`)
      break
    case 'stop':
      showMessage('反转回合顺序！')
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
      gameStatus.value = 'AI的回合'
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
    gameStatus.value = 'AI赢了！'
    showMessage('AI赢得了游戏！')
    setTimeout(() => closeGame(), 3000)
    return
  }

  // Send AI move result to LLM
  // const resultMessage = `${gameStore.character}出了：${card.display}（${getColorText(card.color)} ${card.value}）。${gameStore.character}还剩${aiHand.value.length}张牌。`

  // scriptHandler.sendMessage(resultMessage)

  // Player's turn
  if (aiHand.value.length > 0) {
    isPlayerTurn.value = true
    gameStatus.value = '你的回合'
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
.uno-game {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(30px) saturate(150%);
  -webkit-backdrop-filter: blur(30px) saturate(150%);
  z-index: 9999;
  display: flex;
  flex-direction: column;
  overflow-y: scroll;
  overflow-x: hidden;
}

.uno-game::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background:
    radial-gradient(circle at 10% 20%, rgba(255, 255, 255, 0.08) 0%, transparent 30%),
    radial-gradient(circle at 90% 30%, rgba(255, 255, 255, 0.05) 0%, transparent 40%),
    radial-gradient(circle at 50% 80%, rgba(255, 255, 255, 0.06) 0%, transparent 50%);
  pointer-events: none;
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
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 40px;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(20px) saturate(180%);
  -webkit-backdrop-filter: blur(20px) saturate(180%);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

.game-title {
  font-size: 32px;
  font-weight: bold;
  color: white;
  text-shadow: 0 0 20px rgba(255, 255, 255, 0.5);
}

.close-btn {
  position: absolute;
  right: 0;
  background: rgba(255, 255, 255, 0.2);
  border: 2px solid rgba(255, 255, 255, 0.3);
  color: white;
  width: 100px;
  height: 40px;
  border-radius: 4px;
  font-size: 24px;
  cursor: pointer;
  transition: all 0.3s;
}

.close-btn:hover {
  background: rgba(255, 255, 255, 0.3);
  transform: scale(1.1);
}

/* Avatar Box */
.avatar-box {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  overflow: hidden;
  margin-bottom: 20px;
  border: 4px solid rgba(255, 255, 255, 0.3);
  box-shadow:
    0 8px 25px rgba(0, 0, 0, 0.3),
    inset 0 2px 10px rgba(255, 255, 255, 0.2);
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(10px);
  transition: all 0.3s ease;
}

.avatar-box:hover {
  transform: scale(1.1);
  border-color: rgba(255, 255, 255, 0.5);
  box-shadow:
    0 12px 35px rgba(0, 0, 0, 0.4),
    inset 0 2px 10px rgba(255, 255, 255, 0.3);
}

.avatar-img {
  width: 100%;
  height: 100%;
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  transition: transform 0.3s ease;
}

.avatar-box:hover .avatar-img {
  transform: scale(1.15);
}

/* AI Card Area */
.ai-card-area {
  padding: 20px;
  min-height: 200px;
  display: flex;
  flex-direction: row;
  justify-content: center;
  gap: 100px;
  align-items: center;
  background: rgba(255, 255, 255, 0.02);
}

.player-info {
  display: flex;
  gap: 20px;
  margin-bottom: 15px;
  color: white;
  font-size: 18px;
  align-items: center;
}

.player-name {
  font-weight: bold;
  text-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
}

.card-count,
.player-count {
  color: #ffd700;
}

.ai-cards {
  display: flex;
  position: relative;
  height: 150px;
}

/* Center Area */
.center-area {
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 60px;
  padding: 40px;
}

.draw-pile,
.discard-pile {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 15px;
}

.pile-label {
  color: white;
  font-size: 18px;
  font-weight: bold;
  text-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
}

.card-stack {
  width: 120px;
  height: 180px;
  border-radius: 12px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
  transition: transform 0.3s;
  cursor: pointer;
}

.draw-pile .card-stack:hover {
  transform: translateY(-10px) scale(1.05);
}

.card-stack img {
  width: 100%;
  height: 100%;
  border-radius: 12px;
}

.game-status {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  background: rgba(255, 255, 255, 0.08);
  padding: 20px 30px;
  border-radius: 20px;
  backdrop-filter: saturate(180%);
  -webkit-backdrop-filter: saturate(180%);
  border: 1px solid rgba(255, 255, 255, 0.15);
  box-shadow:
    0 8px 32px rgba(0, 0, 0, 0.2),
    inset 0 1px 2px rgba(255, 255, 255, 0.1);
}

.current-turn {
  color: white;
  font-size: 24px;
  font-weight: bold;
  text-shadow: 0 0 15px rgba(255, 255, 255, 0.7);
}

.current-color {
  font-size: 20px;
  font-weight: bold;
  text-shadow: 0 0 10px currentColor;
}

/* Player Card Area */
.player-card-area {
  padding: 20px;
  min-height: 250px;
  display: flex;
  flex-direction: column;
  align-items: center;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(15px) saturate(150%);
  -webkit-backdrop-filter: blur(15px) saturate(150%);
  border-top: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 -4px 15px rgba(0, 0, 0, 0.1);
}

.player-cards {
  display: flex;
  position: relative;
  height: 180px;
  margin-bottom: 20px;
}

.card {
  position: absolute;
  width: 100px;
  height: 150px;
  border-radius: 10px;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
  transition: all 0.3s ease;
  cursor: pointer;
  z-index: 1;
}

.card img {
  width: 100%;
  height: 100%;
  border-radius: 10px;
  pointer-events: none;
}

.card.playable {
  cursor: pointer;
}

.card.playable:hover {
  transform: translateY(-20px);
  box-shadow: 0 15px 40px rgba(255, 255, 255, 0.4);
  z-index: 100 !important;
}

.card.selected {
  transform: translateY(-20px);
  box-shadow: 0 0 40px rgba(255, 215, 0, 0.8);
  z-index: 101 !important;
}

.card-back {
  cursor: default;
}

.play-card-btn {
  background: linear-gradient(135deg, #8998d8 0%, #f5f6ff 100%);
  color: white;
  border: none;
  padding: 15px 40px;
  font-size: 20px;
  font-weight: bold;
  border-radius: 25px;
  cursor: pointer;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
  transition: all 0.3s;
}

.play-card-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.4);
}

/* Color Picker Modal */
.color-picker-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 10000;
  backdrop-filter: blur(5px);
}

.color-picker-content {
  background: white;
  padding: 40px;
  border-radius: 20px;
  box-shadow: 0 10px 50px rgba(0, 0, 0, 0.5);
}

.color-picker-content h3 {
  margin: 0 0 30px 0;
  font-size: 28px;
  text-align: center;
  color: #333;
}

.color-options {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 20px;
}

.color-option {
  padding: 30px;
  border-radius: 15px;
  font-size: 20px;
  font-weight: bold;
  text-align: center;
  color: white;
  cursor: pointer;
  transition: all 0.3s;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
}

.color-option:hover {
  transform: scale(1.1);
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.4);
}

/* Game Message */
.game-message {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 30px 50px;
  border-radius: 15px;
  font-size: 24px;
  font-weight: bold;
  z-index: 10001;
  backdrop-filter: blur(10px);
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.5);
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s;
}

.fade-enter-from,
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

/* UNO Game Dialog Positioning */
.uno-game-dialog {
  position: fixed !important;
  left: 20px !important;
  top: 50% !important;
  transform: translateY(-50%) !important;
  width: 400px !important;
  max-width: 90vw !important;
  z-index: 10000 !important;
}

/* Loading Overlay */
.loading-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 10002;
  backdrop-filter: blur(5px);
}

.loading-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.loading-spinner {
  width: 60px;
  height: 60px;
  border: 4px solid rgba(255, 255, 255, 0.3);
  border-top: 4px solid #fff;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.loading-text {
  color: white;
  font-size: 24px;
  font-weight: bold;
  text-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
}
</style>
