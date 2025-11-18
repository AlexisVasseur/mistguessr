<script setup>
import { ref, onMounted } from 'vue'
import imageMapping from '../assets/exits/mapping.json'

const emit = defineEmits(['openGallery'])

// Import dynamique LAZY de toutes les images du dossier exits
const mistImagesGlob = import.meta.glob('/src/assets/exits/*.png', { import: 'default' })

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

// Création de la base de données d'images avec mapping des rooms
const imageDatabase = []
const mistImages = [] // Liste des vrais noms pour l'autocomplétion

// Parcourir toutes les rooms et leurs exits
Object.values(imageMapping).forEach((room) => {
  room.exits.forEach((exit) => {
    const imagePath = `/src/assets/exits/${exit.image}`
    const loader = mistImagesGlob[imagePath]

    if (loader) {
      const anonymousId = hashString(exit.image + Date.now())

      imageDatabase.push({
        id: anonymousId,
        name: exit.name,
        loader: loader // Fonction qui charge l'image à la demande
      })

      mistImages.push(exit.name)
    }
  })
})

const currentImageId = ref('')
const currentImageUrl = ref('')
const currentRoomName = ref('')
const userInput = ref('')
const exitCountGuess = ref(null)
const showResult = ref(false)
const isCorrect = ref(false)
const isExitCountCorrect = ref(false)
const correctAnswer = ref('') // Ne sera rempli qu'après validation pour l'affichage
const correctExitCount = ref(0)
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
    correctExitCount.value = 0

    // Détermine la salle à partir du nom de l'exit (ex: "m1_left" -> "m1")
    const exitName = image.name
    const roomName = exitName.split('_')[0]
    currentRoomName.value = roomName

    // Trouve la salle dans le mapping et compte ses exits
    const room = imageMapping[roomName]
    if (room) {
      correctExitCount.value = room.exits.length
    }
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
  if (!userInput.value.trim() || !exitCountGuess.value) return

  // Récupère le vrai nom de l'image depuis la base de données
  const currentImage = imageDatabase.find(img => img.id === currentImageId.value)
  const realName = currentImage.name

  totalAttempts.value++

  // Valide le nom de l'exit
  isCorrect.value = userInput.value.toLowerCase() === realName.toLowerCase()

  // Valide le nombre d'exits
  isExitCountCorrect.value = exitCountGuess.value === correctExitCount.value

  // Le score augmente seulement si les deux réponses sont correctes
  if (isCorrect.value && isExitCountCorrect.value) {
    score.value++
  }

  // Ne révèle les réponses réelles qu'après validation
  correctAnswer.value = realName
  showResult.value = true
  showSuggestions.value = false
}

const nextImage = () => {
  loadRandomImage()
  userInput.value = ''
  exitCountGuess.value = null
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
  </header>

  <div class="game-layout">
    <main class="main-content">
      <div class="image-container">
        <div class="image-wrapper">
          <div v-if="isLoading" class="loading-indicator">Loading...</div>
          <img v-else :src="currentImageUrl" alt="Mist image" class="mist-image" />
        </div>
      </div>
    </main>

    <aside class="controls">
      <div class="score-display">
        Score: {{ score }} / {{ totalAttempts }}
      </div>

      <div class="input-section">
        <div class="input-group">
          <label class="input-label">Exit name:</label>
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
        </div>

        <div class="input-group">
          <label class="input-label">Number of exits in this room:</label>
          <input
            v-model.number="exitCountGuess"
            type="number"
            min="1"
            placeholder="?"
            class="guess-input"
            :disabled="showResult"
          />
        </div>

        <div class="button-group">
          <button @click="validateGuess" :disabled="showResult || !userInput.trim() || !exitCountGuess" class="validate-btn">
            Validate
          </button>
          <button @click="nextImage" :disabled="!showResult" class="next-btn">Next</button>
        </div>
      </div>

      <div class="result-section">
        <div v-show="showResult" class="result-messages">
          <div :class="['result-message', isCorrect ? 'correct' : 'incorrect']">
            <span v-if="isCorrect">✓ Exit name correct!</span>
            <span v-else>✗ Wrong exit! The answer was: {{ correctAnswer }}</span>
          </div>
          <div :class="['result-message', isExitCountCorrect ? 'correct' : 'incorrect']">
            <span v-if="isExitCountCorrect">✓ Exit count correct!</span>
            <span v-else>✗ Wrong count! The room has {{ correctExitCount }} exits</span>
          </div>
        </div>
      </div>
    </aside>
  </div>

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
  font-size: var(--font-size-xl);
  color: var(--color-text-accent);
  font-weight: var(--font-weight-medium);
  text-align: center;
  padding: var(--spacing-md);
  background: var(--color-bg-overlay-dark);
  border-radius: var(--radius-md);
  border: 2px solid var(--color-border-primary);
}

.game-layout {
  display: flex;
  flex: 1;
  min-height: 0;
  overflow: hidden;
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
  width: 400px;
  padding: var(--spacing-xl);
  background: var(--color-bg-overlay-medium);
  border-left: 2px solid var(--color-border-primary);
  display: flex;
  flex-direction: column;
  gap: var(--spacing-xl);
  flex-shrink: 0;
  overflow-y: auto;
}

.input-section {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-md);
  width: 100%;
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-xs);
}

.input-label {
  font-size: var(--font-size-sm);
  color: var(--color-text-secondary);
  font-weight: var(--font-weight-medium);
  text-transform: uppercase;
  letter-spacing: var(--letter-spacing-sm);
}

.button-group {
  display: flex;
  gap: var(--spacing-md);
  width: 100%;
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
  top: 100%;
  left: 0;
  right: 0;
  background: var(--color-bg-suggestion);
  border: 2px solid var(--color-border-secondary);
  border-top: none;
  border-radius: 0 0 var(--radius-md) var(--radius-md);
  max-height: var(--suggestions-max-height);
  overflow-y: auto;
  list-style: none;
  margin: 0;
  margin-top: var(--spacing-xs);
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
  flex: 1;
  padding: var(--spacing-md) var(--spacing-lg);
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

.validate-btn:hover:not(:disabled),
.next-btn:hover:not(:disabled) {
  background: var(--gradient-button-hover);
  box-shadow: var(--shadow-glow-strong);
  transform: translateY(-2px);
}

.validate-btn:disabled,
.next-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.result-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 100%;
}

.result-messages {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-sm);
  width: 100%;
}

.result-message {
  padding: var(--spacing-sm) var(--spacing-lg);
  border-radius: var(--radius-md);
  font-size: var(--font-size-sm);
  font-weight: var(--font-weight-medium);
  width: 100%;
  text-align: center;
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
  top:15px;
  left: 15px;
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

  .game-layout {
    flex-direction: column;
  }

  .main-content {
    padding: var(--spacing-xs);
  }

  .controls {
    width: 100%;
    border-left: none;
    border-top: 2px solid var(--color-border-primary);
    padding: var(--spacing-md);
    gap: var(--spacing-md);
  }

  .gallery-btn {
    top: var(--spacing-md);
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
