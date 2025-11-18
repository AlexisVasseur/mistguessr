<script setup>
import { onMounted, onUnmounted } from 'vue'

const props = defineProps({
  image: {
    type: Object,
    required: true
  }
})

const emit = defineEmits(['close'])

// Fermer avec la touche Escape
const handleKeydown = (e) => {
  if (e.key === 'Escape') {
    emit('close')
  }
}

onMounted(() => {
  document.addEventListener('keydown', handleKeydown)
  document.body.style.overflow = 'hidden'
})

onUnmounted(() => {
  document.removeEventListener('keydown', handleKeydown)
  document.body.style.overflow = ''
})
</script>

<template>
  <div class="modal-overlay" @click="emit('close')">
    <div class="modal-content">
      <button @click="emit('close')" class="close-btn" aria-label="Close">
        <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <line x1="18" y1="6" x2="6" y2="18"></line>
          <line x1="6" y1="6" x2="18" y2="18"></line>
        </svg>
      </button>

      <div class="image-container">
        <img :src="image.url" :alt="image.name" class="modal-image" @click.stop />
      </div>

      <div class="image-title">{{ image.name }}</div>
    </div>
  </div>
</template>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: var(--color-bg-modal);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: var(--z-modal);
  animation: fadeIn var(--transition-fast) ease-out;
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.modal-content {
  position: relative;
  width: 95vw;
  height: 95vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  animation: scaleIn var(--transition-normal) ease-out;
}

@keyframes scaleIn {
  from {
    transform: scale(0.9);
    opacity: 0;
  }
  to {
    transform: scale(1);
    opacity: 1;
  }
}

.close-btn {
  position: absolute;
  top: var(--spacing-md);
  right: var(--spacing-md);
  width: 48px;
  height: 48px;
  background: var(--color-bg-input);
  border: 2px solid var(--color-border-accent);
  border-radius: var(--radius-circle);
  color: var(--color-border-accent);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all var(--transition-normal);
  z-index: var(--z-close-btn);
}

.close-btn:hover {
  background: var(--color-success-bg);
  box-shadow: var(--shadow-glow-intense);
  transform: rotate(90deg);
}

.image-container {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  max-height: calc(95vh - 80px);
}

.modal-image {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  border: 3px solid var(--color-border-secondary);
  border-radius: var(--radius-lg);
  box-shadow: var(--shadow-lg);
}

.image-title {
  margin-top: var(--spacing-lg);
  font-size: var(--font-size-xl);
  color: var(--color-text-accent);
  text-shadow: var(--shadow-glow);
  font-weight: var(--font-weight-light);
  letter-spacing: var(--letter-spacing-md);
  text-align: center;
}

@media (max-width: 768px) {
  .close-btn {
    width: 40px;
    height: 40px;
    top: var(--spacing-xs);
    right: var(--spacing-xs);
  }

  .image-title {
    font-size: var(--font-size-lg);
    margin-top: var(--spacing-md);
  }
}
</style>
