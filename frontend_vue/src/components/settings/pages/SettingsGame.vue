<template>
  <MenuPage>
    <MenuItem title="休闲游戏">
      <div class="game-list">
        <div class="game-item">
          <img src="/pictures/games/uno/game_avatar.png" alt="UNO" class="game-icon" />
          <div class="game-info">
            <h3>UNO</h3>
            <p>经典的UNO卡牌游戏</p>
          </div>
          <Button type="nav" @click="playUNO">开始游戏</Button>
        </div>
      </div>
    </MenuItem>
    <!-- UNO Game Component -->
    <UnoGame ref="unoGameRef" />
  </MenuPage>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import MenuPage from '@/components/ui/Menu/MenuPage.vue'
import { Button } from '../../base'
import MenuItem from '@/components/ui/Menu/MenuItem.vue'
import UnoGame from '@/components/game/uno/UnoGame.vue'

const unoGameRef = ref<InstanceType<typeof UnoGame> | null>(null)

const playUNO = () => {
  console.log('启动UNO游戏')
  if (unoGameRef.value) {
    unoGameRef.value.startGame()
  }
}
</script>

<style scoped>
@reference "tailwindcss";

.settings-page {
  @apply text-[white] p-5;
}
.settings-title {
  @apply text-2xl mb-5;
}
.game-list {
  @apply flex flex-col gap-5 grid-cols-[repeat(auto-fill,minmax(320px,1fr))];
}
.game-item {
  @apply flex items-center gap-[15px] p-[15px] rounded-lg;
  background: rgba(255, 255, 255, 0.1);
}
.game-icon {
  @apply w-[60px] h-[60px] rounded-lg;
}
.game-info {
  @apply flex-1;
}
.game-info h3 {
  @apply text-lg m-0;
}
.game-info p {
  @apply text-[#ccc] mt-[5px] mb-0 mx-0;
}
</style>
