<script setup>
import { ref, onMounted } from 'vue'
import imageMapping from './mists/mapping.json'

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
  <div class="app">
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
  </div>
</template>

<style scoped>
.app {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: linear-gradient(180deg, #1a2332 0%, #0d1621 100%);
  color: #a8c7d7;
  overflow: hidden;
}

.header {
  padding: 1rem 2rem;
  text-align: center;
  background: rgba(10, 18, 28, 0.5);
  border-bottom: 2px solid #2d4a5a;
  flex-shrink: 0;
}

.header h1 {
  margin: 0 0 0.5rem 0;
  font-size: 2rem;
  color: #6fb3d2;
  text-shadow: 0 0 10px rgba(111, 179, 210, 0.3);
  font-weight: 300;
  letter-spacing: 2px;
}

.score-display {
  font-size: 1.2rem;
  color: #8fc4d9;
  font-weight: 500;
}

.main-content {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1rem;
  overflow: hidden;
  box-sizing: border-box;
  min-height: 0;
}

.image-container {
  background: rgba(13, 22, 33, 0.6);
  border: 3px solid #3d5a6a;
  border-radius: 8px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4);
  width: 100%;
  height: 100%;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1rem;
}

.image-wrapper {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(0, 0, 0, 0.2);
  border-radius: 4px;
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
  color: #6fb3d2;
  font-size: 1.5rem;
  font-weight: 300;
  letter-spacing: 2px;
  text-shadow: 0 0 10px rgba(111, 179, 210, 0.5);
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
  padding: 1rem 2rem;
  background: rgba(10, 18, 28, 0.7);
  border-top: 2px solid #2d4a5a;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  flex-shrink: 0;
}

.input-section {
  display: flex;
  gap: 1rem;
  max-width: 1000px;
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
  padding: 0.75rem;
  font-size: 1rem;
  background: rgba(26, 35, 50, 0.8);
  border: 2px solid #3d5a6a;
  border-radius: 6px;
  color: #a8c7d7;
  outline: none;
  transition: all 0.3s;
}

.guess-input:focus {
  border-color: #6fb3d2;
  box-shadow: 0 0 10px rgba(111, 179, 210, 0.3);
}

.guess-input:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.guess-input::placeholder {
  color: #5a7485;
}

.suggestions-list {
  position: absolute;
  bottom: 100%;
  left: 0;
  right: 0;
  background: rgba(26, 35, 50, 0.98);
  border: 2px solid #3d5a6a;
  border-bottom: none;
  border-radius: 6px 6px 0 0;
  max-height: 200px;
  overflow-y: auto;
  list-style: none;
  margin: 0;
  margin-bottom: 0.5rem;
  padding: 0;
  z-index: 1000;
}

.suggestion-item {
  padding: 0.8rem 1rem;
  cursor: pointer;
  color: #a8c7d7;
  transition: background 0.2s;
}

.suggestion-item:hover,
.suggestion-item.selected {
  background: rgba(111, 179, 210, 0.2);
}

.validate-btn,
.next-btn {
  padding: 0.75rem 1.5rem;
  font-size: 1rem;
  background: linear-gradient(135deg, #4a7c94 0%, #3d5a6a 100%);
  border: 2px solid #6fb3d2;
  border-radius: 6px;
  color: #e0f0f7;
  cursor: pointer;
  transition: all 0.3s;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 1px;
  min-width: 100px;
  flex-shrink: 0;
}

.validate-btn:hover:not(:disabled),
.next-btn:hover {
  background: linear-gradient(135deg, #5a8ca4 0%, #4d6a7a 100%);
  box-shadow: 0 0 15px rgba(111, 179, 210, 0.4);
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
  height: 65px;
  max-width: 1000px;
  margin: 0 auto;
  width: 100%;
}

.result-message {
  padding: 0.75rem 1.5rem;
  border-radius: 6px;
  font-size: 1.1rem;
  font-weight: 500;
  width: 100%;
}

.result-message.correct {
  background: rgba(111, 179, 210, 0.2);
  border: 2px solid #6fb3d2;
  color: #6fb3d2;
}

.result-message.incorrect {
  background: rgba(143, 82, 82, 0.2);
  border: 2px solid #a85858;
  color: #d89090;
}

@media (max-width: 768px) {
  .header h1 {
    font-size: 1.5rem;
  }

  .header {
    padding: 0.75rem 1rem;
  }

  .main-content {
    padding: 0.5rem;
  }

  .input-section {
    flex-direction: column;
    gap: 0.5rem;
  }

  .controls {
    padding: 0.75rem 1rem;
  }

  .validate-btn,
  .next-btn {
    width: 100%;
  }
}
</style>
