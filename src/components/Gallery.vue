<script setup>
import { ref, onMounted } from 'vue'
import imageMapping from '../assets/exits/mapping.json'
import ImageModal from './ImageModal.vue'

const emit = defineEmits(['close'])

// Import lazy des images
const mistImagesGlob = import.meta.glob('/src/assets/exits/*.png', { import: 'default' })
const roomImagesGlob = import.meta.glob('/src/assets/rooms/*.png', { import: 'default' })

// Liste des salles avec leurs images
const roomGroups = ref([])
const isLoading = ref(true)
const selectedImage = ref(null)

// Charger toutes les images groupées par salle
onMounted(async () => {
  const groups = []

  // Parcourir toutes les rooms
  for (const [roomName, room] of Object.entries(imageMapping)) {
    const imagePromises = []

    // Charger l'image de la room
    const roomImagePath = `/src/assets/rooms/${room.room}`
    const roomLoader = roomImagesGlob[roomImagePath]
    let roomImageUrl = null

    if (roomLoader) {
      roomImageUrl = await roomLoader()
    }

    // Charger les exits
    room.exits.forEach((exit) => {
      const imagePath = `/src/assets/exits/${exit.image}`
      const loader = mistImagesGlob[imagePath]

      if (loader) {
        imagePromises.push(
          loader().then(url => ({ name: exit.name, url }))
        )
      }
    })

    const loadedImages = await Promise.all(imagePromises)
    groups.push({
      roomName: roomName,
      roomImage: roomImageUrl,
      images: loadedImages.sort((a, b) => a.name.localeCompare(b.name))
    })
  }

  // Trier les groupes par nom de salle
  roomGroups.value = groups.sort((a, b) => a.roomName.localeCompare(b.roomName))
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

    <div v-else class="gallery-content">
      <div v-for="group in roomGroups" :key="group.roomName" class="room-section">
        <h2 class="room-title">{{ group.roomName }}</h2>

        <div class="gallery-grid">
          <!-- Room image as first item -->
          <div
            v-if="group.roomImage"
            @click="openModal({ name: `${group.roomName} (complete room)`, url: group.roomImage })"
            class="gallery-item room-item"
          >
            <div class="image-wrapper">
              <img :src="group.roomImage" :alt="`${group.roomName} complete room`" class="gallery-image" />
            </div>
            <div class="image-name">{{ group.roomName }} (complete)</div>
          </div>

          <!-- Exits -->
          <div
            v-for="image in group.images"
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

.gallery-content {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-xl);
}

.room-section {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-md);
}

.room-title {
  font-size: var(--font-size-xl);
  color: var(--color-text-accent);
  text-shadow: var(--shadow-glow);
  font-weight: var(--font-weight-medium);
  letter-spacing: var(--letter-spacing-md);
  text-transform: uppercase;
  margin: 0;
  padding-bottom: var(--spacing-sm);
  border-bottom: 2px solid var(--color-border-secondary);
}

.gallery-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: var(--spacing-xl);
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

.room-item .image-wrapper {
  border-color: var(--color-border-accent);
}

.gallery-item:hover .image-wrapper {
  border-color: var(--color-border-accent);
  box-shadow: var(--shadow-glow-strong);
}

.room-item:hover .image-wrapper {
  border-color: var(--color-text-accent);
  box-shadow: var(--shadow-glow-intense);
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
