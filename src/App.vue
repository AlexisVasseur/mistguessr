<script setup>
import { ref } from 'vue'
import Game from './components/Game.vue'
import Gallery from './components/Gallery.vue'

const showGallery = ref(false)
</script>

<template>
  <div class="app" :class="{ 'no-scroll': !showGallery }">
    <Game v-if="!showGallery" @open-gallery="showGallery = true" />
    <Gallery v-else @close="showGallery = false" />
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  background: var(--gradient-bg);
  color: var(--color-text-primary);
}

.app.no-scroll {
  height: 100vh;
  overflow: hidden;
}
</style>
