<script setup>
import { ref, onMounted } from 'vue'
import imageMapping from '../mists/mapping.json'

const emit = defineEmits(['openGallery'])

// Import dynamique LAZY de toutes les images du dossier mists
const mistImagesGlob = import.meta.glob('/src/mists/*.png', { import: 'default' })

// Fonction de hash simple pour générer des IDs uniques
const hashString = (str) => {
  let hash = 0
  for (let i = 0; i < str.length; i++) {
    const char = str.charCodeAt(i)
    hash = ((hash << 5) - hash) + char
    hash = hash & hash
  }
  return Math.abs(hash).toString(36)
}

// Création de la base de données d'images avec mapping
const imageDatabase = []
const mistImages = [] // Liste des vrais noms pour l'autocomplétion

Object.entries(mistImagesGlob).forEach(([path, loader]) => {
  const anonymousFileName = path.split('/').pop() // ex: a7f3d2e9.png
  const realName = imageMapping[anonymousFileName] // ex: m1_left

  if (realName) {
    const anonymousId = hashString(anonymousFileName + Date.now())

    imageDatabase.push({
      id: anonymousId,
      name: realName,
      loader: loader // Fonction qui charge l'image à la demande
    })

    mistImages.push(realName)
  }
})

const currentImageId = ref('')
const currentImageUrl = ref('')
const userInput = ref('')
const showResult = ref(false)
const isCorrect = ref(false)
const correctAnswer = ref('') // Ne sera rempli qu'après validation pour l'affichage
const score = ref(0)
const totalAttempts = ref(0)
const filteredSuggestions = ref([])
const showSuggestions = ref(false)
const selectedIndex = ref(-1)
const isLoading = ref(false)

const loadRandomImage = async () => {
  isLoading.value = true
  const randomIndex = Math.floor(Math.random() * imageDatabase.length)
  const image = imageDatabase[randomIndex]

  try {
    // Charge l'image de manière lazy
    const url = await image.loader()
    currentImageId.value = image.id
    currentImageUrl.value = url
    correctAnswer.value = '' // Réinitialise pour cacher la réponse
  } catch (error) {
    console.error('Erreur lors du chargement de l\'image:', error)
  } finally {
    isLoading.value = false
  }
}

const filterSuggestions = () => {
  if (userInput.value.length > 0) {
    filteredSuggestions.value = mistImages.filter(img =>
      img.toLowerCase().includes(userInput.value.toLowerCase())
    )
    showSuggestions.value = true
    selectedIndex.value = -1
  } else {
    filteredSuggestions.value = []
    showSuggestions.value = false
  }
}

const selectSuggestion = (suggestion) => {
  userInput.value = suggestion
  showSuggestions.value = false
  selectedIndex.value = -1
}

const handleKeydown = (event) => {
  if (!showSuggestions.value || filteredSuggestions.value.length === 0) return

  if (event.key === 'ArrowDown') {
    event.preventDefault()
    selectedIndex.value = Math.min(selectedIndex.value + 1, filteredSuggestions.value.length - 1)
  } else if (event.key === 'ArrowUp') {
    event.preventDefault()
    selectedIndex.value = Math.max(selectedIndex.value - 1, -1)
  } else if (event.key === 'Enter' && selectedIndex.value >= 0) {
    event.preventDefault()
    selectSuggestion(filteredSuggestions.value[selectedIndex.value])
  }
}

const validateGuess = () => {
  if (!userInput.value.trim()) return

  // Récupère le vrai nom de l'image depuis la base de données
  const currentImage = imageDatabase.find(img => img.id === currentImageId.value)
  const realName = currentImage.name

  totalAttempts.value++
  isCorrect.value = userInput.value.toLowerCase() === realName.toLowerCase()

  if (isCorrect.value) {
    score.value++
  }

  // Ne révèle le nom réel qu'après validation
  correctAnswer.value = realName
  showResult.value = true
  showSuggestions.value = false
}

const nextImage = () => {
  loadRandomImage()
  userInput.value = ''
  showResult.value = false
  selectedIndex.value = -1
}

onMounted(() => {
  loadRandomImage()
})
</script>

<template>
  <header class="header">
    <h1>Silksong Mistguessr</h1>
    <div class="score-display">
      Score: {{ score }} / {{ totalAttempts }}
    </div>
  </header>

  <main class="main-content">
    <div class="image-container">
      <div class="image-wrapper">
        <div v-if="isLoading" class="loading-indicator">Loading...</div>
        <img v-else :src="currentImageUrl" alt="Mist image" class="mist-image" />
      </div>
    </div>
  </main>

  <footer class="controls">
    <div class="input-section">
      <button @click="nextImage" class="next-btn">Next</button>
      <div class="autocomplete-wrapper">
        <input
          v-model="userInput"
          @input="filterSuggestions"
          @keydown="handleKeydown"
          @focus="filterSuggestions"
          type="text"
          placeholder="Type the mist name..."
          class="guess-input"
          :disabled="showResult"
        />
        <ul v-if="showSuggestions && filteredSuggestions.length > 0" class="suggestions-list">
          <li
            v-for="(suggestion, index) in filteredSuggestions"
            :key="suggestion"
            @click="selectSuggestion(suggestion)"
            :class="{ 'selected': index === selectedIndex }"
            class="suggestion-item"
          >
            {{ suggestion }}
          </li>
        </ul>
      </div>
      <button @click="validateGuess" :disabled="showResult || !userInput.trim()" class="validate-btn">
        Validate
      </button>
    </div>

    <div class="result-section">
      <div v-show="showResult" :class="['result-message', isCorrect ? 'correct' : 'incorrect']">
        <span v-if="isCorrect">✓ Correct!</span>
        <span v-else>✗ Wrong! The answer was: {{ correctAnswer }}</span>
      </div>
    </div>
  </footer>

  <!-- Bouton galerie flottant -->
  <button @click="emit('openGallery')" class="gallery-btn" title="Open Gallery">
    <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
      <rect x="3" y="3" width="7" height="7"></rect>
      <rect x="14" y="3" width="7" height="7"></rect>
      <rect x="14" y="14" width="7" height="7"></rect>
      <rect x="3" y="14" width="7" height="7"></rect>
    </svg>
  </button>
</template>

<style scoped>
.header {
  padding: var(--spacing-md) var(--spacing-xl);
  text-align: center;
  background: var(--color-bg-overlay-dark);
  border-bottom: 2px solid var(--color-border-primary);
  flex-shrink: 0;
}

.header h1 {
  margin: 0 0 var(--spacing-xs) 0;
  font-size: var(--font-size-2xl);
  color: var(--color-text-accent);
  text-shadow: var(--shadow-glow);
  font-weight: var(--font-weight-light);
  letter-spacing: var(--letter-spacing-md);
}

.score-display {
  font-size: var(--font-size-lg);
  color: var(--color-text-secondary);
  font-weight: var(--font-weight-medium);
}

.main-content {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: var(--spacing-md);
  overflow: hidden;
  box-sizing: border-box;
  min-height: 0;
}

.image-container {
  background: var(--color-bg-card);
  border: 3px solid var(--color-border-secondary);
  border-radius: var(--radius-lg);
  box-shadow: var(--shadow-md);
  width: 100%;
  height: 100%;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: var(--spacing-md);
}

.image-wrapper {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--color-bg-image-wrapper);
  border-radius: var(--radius-sm);
  overflow: hidden;
}

.mist-image {
  max-width: 100%;
  max-height: 100%;
  width: auto;
  height: auto;
  object-fit: contain;
  display: block;
}

.loading-indicator {
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--color-text-accent);
  font-size: var(--font-size-xl);
  font-weight: var(--font-weight-light);
  letter-spacing: var(--letter-spacing-md);
  text-shadow: var(--shadow-glow-strong);
  animation: pulse 1.5s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% {
    opacity: 0.6;
  }
  50% {
    opacity: 1;
  }
}

.controls {
  padding: var(--spacing-md) var(--spacing-xl);
  background: var(--color-bg-overlay-medium);
  border-top: 2px solid var(--color-border-primary);
  display: flex;
  flex-direction: column;
  gap: var(--spacing-sm);
  flex-shrink: 0;
}

.input-section {
  display: flex;
  gap: var(--spacing-md);
  max-width: var(--input-max-width);
  margin: 0 auto;
  width: 100%;
  align-items: center;
}

.autocomplete-wrapper {
  flex: 1;
  position: relative;
}

.guess-input {
  width: 100%;
  padding: var(--spacing-sm);
  font-size: var(--font-size-sm);
  background: var(--color-bg-input);
  border: 2px solid var(--color-border-secondary);
  border-radius: var(--radius-md);
  color: var(--color-text-primary);
  outline: none;
  transition: all var(--transition-normal);
}

.guess-input:focus {
  border-color: var(--color-border-accent);
  box-shadow: var(--shadow-glow);
}

.guess-input:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.guess-input::placeholder {
  color: var(--color-text-placeholder);
}

.suggestions-list {
  position: absolute;
  bottom: 100%;
  left: 0;
  right: 0;
  background: var(--color-bg-suggestion);
  border: 2px solid var(--color-border-secondary);
  border-bottom: none;
  border-radius: var(--radius-md) var(--radius-md) 0 0;
  max-height: var(--suggestions-max-height);
  overflow-y: auto;
  list-style: none;
  margin: 0;
  margin-bottom: var(--spacing-xs);
  padding: 0;
  z-index: var(--z-suggestions);
}

.suggestion-item {
  padding: 0.8rem var(--spacing-md);
  cursor: pointer;
  color: var(--color-text-primary);
  transition: background var(--transition-fast);
}

.suggestion-item:hover,
.suggestion-item.selected {
  background: var(--color-success-bg);
}

.validate-btn,
.next-btn {
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
  min-width: var(--button-width-min);
  flex-shrink: 0;
}

.validate-btn:hover:not(:disabled),
.next-btn:hover {
  background: var(--gradient-button-hover);
  box-shadow: var(--shadow-glow-strong);
  transform: translateY(-2px);
}

.validate-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.result-section {
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: var(--result-height);
  max-width: var(--input-max-width);
  margin: 0 auto;
  width: 100%;
}

.result-message {
  padding: var(--spacing-sm) var(--spacing-lg);
  border-radius: var(--radius-md);
  font-size: var(--font-size-md);
  font-weight: var(--font-weight-medium);
  width: 100%;
}

.result-message.correct {
  background: var(--color-success-bg);
  border: 2px solid var(--color-success-border);
  color: var(--color-success-text);
}

.result-message.incorrect {
  background: var(--color-error-bg);
  border: 2px solid var(--color-error-border);
  color: var(--color-error-text);
}

.gallery-btn {
  position: fixed;
  bottom: var(--spacing-xl);
  left: var(--spacing-xl);
  width: var(--button-height-gallery);
  height: var(--button-height-gallery);
  background: var(--gradient-button);
  border: 2px solid var(--color-border-accent);
  border-radius: var(--radius-circle);
  color: var(--color-text-light);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all var(--transition-normal);
  box-shadow: var(--shadow-sm);
  z-index: var(--z-gallery-btn);
}

.gallery-btn:hover {
  background: var(--gradient-button-hover);
  box-shadow: var(--shadow-glow-intense);
  transform: translateY(-3px);
}

.gallery-btn svg {
  width: 28px;
  height: 28px;
}

@media (max-width: 768px) {
  .header h1 {
    font-size: var(--font-size-xl);
  }

  .header {
    padding: var(--spacing-sm) var(--spacing-md);
  }

  .main-content {
    padding: var(--spacing-xs);
  }

  .input-section {
    flex-direction: column;
    gap: var(--spacing-xs);
  }

  .controls {
    padding: var(--spacing-sm) var(--spacing-md);
  }

  .validate-btn,
  .next-btn {
    width: 100%;
  }

  .gallery-btn {
    bottom: var(--spacing-md);
    left: var(--spacing-md);
    width: var(--button-height-gallery-mobile);
    height: var(--button-height-gallery-mobile);
  }

  .gallery-btn svg {
    width: 24px;
    height: 24px;
  }
}
</style>
