<script setup>
import { ref, onMounted } from 'vue'
import imageMapping from '../mists/mapping.json'
import ImageModal from './ImageModal.vue'

const emit = defineEmits(['close'])

// Import lazy des images
const mistImagesGlob = import.meta.glob('/src/mists/*.png', { import: 'default' })

// Liste des images avec leurs infos
const images = ref([])
const isLoading = ref(true)
const selectedImage = ref(null)

// Charger toutes les images
onMounted(async () => {
  const imagePromises = Object.entries(mistImagesGlob).map(async ([path, loader]) => {
    const anonymousFileName = path.split('/').pop()
    const realName = imageMapping[anonymousFileName]

    if (realName) {
      const url = await loader()
      return { name: realName, url }
    }
    return null
  })

  const loadedImages = await Promise.all(imagePromises)
  images.value = loadedImages.filter(img => img !== null).sort((a, b) => a.name.localeCompare(b.name))
  isLoading.value = false
})

const openModal = (image) => {
  selectedImage.value = image
}

const closeModal = () => {
  selectedImage.value = null
}
</script>

<template>
  <div class="gallery-container">
    <header class="gallery-header">
      <h1>Mist Gallery</h1>
      <button @click="emit('close')" class="back-btn">Back to Game</button>
    </header>

    <div v-if="isLoading" class="loading">Loading gallery...</div>

    <div v-else class="gallery-grid">
      <div
        v-for="image in images"
        :key="image.name"
        @click="openModal(image)"
        class="gallery-item"
      >
        <div class="image-wrapper">
          <img :src="image.url" :alt="image.name" class="gallery-image" />
        </div>
        <div class="image-name">{{ image.name }}</div>
      </div>
    </div>

    <ImageModal
      v-if="selectedImage"
      :image="selectedImage"
      @close="closeModal"
    />
  </div>
</template>

<style scoped>
.gallery-container {
  min-height: 100vh;
  background: var(--gradient-bg);
  color: var(--color-text-primary);
  padding: var(--spacing-xl);
}

.gallery-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--spacing-xl);
  padding-bottom: var(--spacing-md);
  border-bottom: 2px solid var(--color-border-primary);
}

.gallery-header h1 {
  font-size: var(--font-size-2xl);
  color: var(--color-text-accent);
  text-shadow: var(--shadow-glow);
  font-weight: var(--font-weight-light);
  letter-spacing: var(--letter-spacing-md);
  margin: 0;
}

.back-btn {
  padding: var(--spacing-sm) var(--spacing-lg);
  font-size: var(--font-size-sm);
  background: var(--gradient-button);
  border: 2px solid var(--color-border-accent);
  border-radius: var(--radius-md);
  color: var(--color-text-light);
  cursor: pointer;
  transition: all var(--transition-normal);
  font-weight: var(--font-weight-medium);
  text-transform: uppercase;
  letter-spacing: var(--letter-spacing-sm);
}

.back-btn:hover {
  background: var(--gradient-button-hover);
  box-shadow: var(--shadow-glow-strong);
  transform: translateY(-2px);
}

.loading {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 50vh;
  font-size: var(--font-size-xl);
  color: var(--color-text-accent);
  animation: pulse 1.5s ease-in-out infinite;
}

.gallery-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: var(--spacing-xl);
  padding: var(--spacing-md) 0;
}

.gallery-item {
  cursor: pointer;
  transition: transform var(--transition-normal);
}

.gallery-item:hover {
  transform: translateY(-5px);
}

.image-wrapper {
  background: var(--color-bg-card);
  border: 3px solid var(--color-border-secondary);
  border-radius: var(--radius-lg);
  padding: var(--spacing-md);
  display: flex;
  align-items: center;
  justify-content: center;
  height: 200px;
  transition: border-color var(--transition-normal);
}

.gallery-item:hover .image-wrapper {
  border-color: var(--color-border-accent);
  box-shadow: var(--shadow-glow-strong);
}

.gallery-image {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}

.image-name {
  text-align: center;
  margin-top: var(--spacing-sm);
  font-size: var(--font-size-sm);
  color: var(--color-text-primary);
  font-weight: var(--font-weight-normal);
}

@keyframes pulse {
  0%, 100% {
    opacity: 0.6;
  }
  50% {
    opacity: 1;
  }
}

@media (max-width: 768px) {
  .gallery-container {
    padding: var(--spacing-md);
  }

  .gallery-grid {
    grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
    gap: var(--spacing-md);
  }

  .image-wrapper {
    height: 150px;
  }
}
</style>
