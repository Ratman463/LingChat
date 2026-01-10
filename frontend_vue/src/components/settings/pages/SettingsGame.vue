<template>
  <div class="settings-page">
    <h2 class="settings-title">游戏设置</h2>
    <div class="game-list">
      <div class="game-item">
        <img src="/pictures/games/uno/game_avatar.png" alt="UNO" class="game-icon" />
        <div class="game-info">
          <h3>UNO</h3>
          <p>经典的UNO卡牌游戏</p>
        </div>
        <Button type="big" @click="playUNO">开始游戏</Button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { Button } from '../../base'

const playUNO = () => {
  // TODO: 实现UNO游戏逻辑
  console.log('启动UNO游戏')
}
</script>

<style scoped>
.settings-page {
  padding: 20px;
  color: white;
}

.settings-title {
  font-size: 24px;
  margin-bottom: 20px;
}

.game-list {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.game-item {
  display: flex;
  align-items: center;
  gap: 15px;
  padding: 15px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 8px;
}

.game-icon {
  width: 60px;
  height: 60px;
  border-radius: 8px;
}

.game-info {
  flex: 1;
}

.game-info h3 {
  margin: 0;
  font-size: 18px;
}

.game-info p {
  margin: 5px 0 0 0;
  color: #ccc;
}
</style>
