<template>
  <div class="app">
    <!-- Vista de Login/Registro -->
    <div v-if="!user" class="auth-view">
      <div class="auth-container">
        <h1 class="auth-logo">📚 Flashcards</h1>
        <p class="auth-subtitle">Organiza tus tarjetas de estudio</p>

        <div class="auth-tabs">
          <button 
            @click="authMode = 'login'" 
            class="auth-tab"
            :class="{ active: authMode === 'login' }"
          >
            Iniciar Sesión
          </button>
          <button 
            @click="authMode = 'register'" 
            class="auth-tab"
            :class="{ active: authMode === 'register' }"
          >
            Registrarse
          </button>
        </div>

        <form @submit.prevent="authMode === 'login' ? login() : register()" class="auth-form">
          <input
            v-model="authEmail"
            type="email"
            placeholder="Email"
            class="input"
            required
          />
          <input
            v-model="authPassword"
            type="password"
            placeholder="Contraseña"
            class="input"
            required
            :minlength="6"
          />
          
          <button 
            type="submit" 
            class="btn-primary btn-block"
            :disabled="isAuthLoading"
          >
            <span v-if="isAuthLoading" class="spinner"></span>
            <span v-else>{{ authMode === 'login' ? 'Iniciar Sesión' : 'Registrarse' }}</span>
          </button>

          <p v-if="authError" class="error-message">{{ authError }}</p>
        </form>
      </div>
    </div>

    <!-- Vista Principal (cuando está autenticado) -->
    <div v-else>
      <header class="header">
        <div class="container header-content">
          <h1 class="logo">📚 Flashcards</h1>
          <div class="user-section">
            <span class="user-email">{{ user.email }}</span>
            <button @click="logout" class="btn-logout">Cerrar Sesión</button>
          </div>
        </div>
      </header>

      <main class="container">
        <div v-if="!selectedTopic" class="topics-view">
          <div class="view-header">
            <h2>Mis Temas</h2>
            <button @click="showNewTopicModal = true" class="btn-primary" :disabled="isLoadingTopics">
              + Nuevo Tema
            </button>
          </div>

          <!-- Skeleton loading -->
          <div v-if="isLoadingTopics" class="topics-grid">
            <div class="skeleton-card" v-for="i in 3" :key="i"></div>
          </div>

          <div v-else-if="topics.length === 0" class="empty-state">
            <p>No hay temas aún. ¡Crea tu primer tema!</p>
          </div>

          <div v-else class="topics-grid">
            <div
              v-for="topic in topics"
              :key="topic.id"
              @click="selectTopic(topic)"
              class="topic-card"
            >
              <div class="topic-header">
                <h3>{{ topic.name }}</h3>
                <button
                  @click.stop="deleteTopic(topic.id)"
                  class="btn-icon"
                  title="Eliminar tema"
                  :disabled="isDeletingTopic === topic.id"
                >
                  <span v-if="isDeletingTopic === topic.id" class="spinner"></span>
                  <span v-else>🗑️</span>
                </button>
              </div>
              <p class="topic-count">{{ topic.flashcardCount || 0 }} flashcards</p>
            </div>
          </div>
        </div>

        <div v-else class="flashcards-view">
          <div class="view-header">
            <button @click="selectedTopic = null" class="btn-back" :disabled="isLoadingFlashcards">
              ← Volver
            </button>
            <h2>{{ selectedTopic.name }}</h2>
            <button @click="showNewCardModal = true" class="btn-primary" :disabled="isLoadingFlashcards">
              + Nueva Flashcard
            </button>
          </div>

          <!-- Skeleton loading -->
          <div v-if="isLoadingFlashcards" class="flashcards-grid">
            <div class="skeleton-card skeleton-flashcard" v-for="i in 3" :key="i"></div>
          </div>

          <div v-else-if="flashcards.length === 0" class="empty-state">
            <p>No hay flashcards aún. ¡Crea tu primera flashcard!</p>
          </div>

          <div v-else class="flashcards-grid">
            <div
              v-for="card in flashcards"
              :key="card.id"
              class="flashcard"
              :class="{ flipped: card.isFlipped }"
              @click="toggleFlip(card)"
            >
              <div class="flashcard-inner">
                <div class="flashcard-front">
                  <p>{{ card.question }}</p>
                  <span class="flip-hint">Click para ver respuesta</span>
                </div>
                <div class="flashcard-back">
                  <p>{{ card.answer }}</p>
                  <button
                    @click.stop="deleteFlashcard(card.id)"
                    class="btn-delete"
                    :disabled="isDeletingCard === card.id"
                  >
                    <span v-if="isDeletingCard === card.id" class="spinner"></span>
                    <span v-else>Eliminar</span>
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>
      </main>
    </div>

    <!-- Modal Nuevo Tema -->
    <div v-if="showNewTopicModal" class="modal-overlay" @click="closeTopicModal">
      <div class="modal" @click.stop>
        <h3>Nuevo Tema</h3>
        <input
          v-model="newTopicName"
          @keyup.enter="createTopic"
          type="text"
          placeholder="Ej: AWS, JavaScript, Historia..."
          class="input"
          autofocus
        />
        <div class="modal-actions">
          <button @click="closeTopicModal" class="btn-secondary" :disabled="isCreatingTopic">
            Cancelar
          </button>
          <button @click="createTopic" class="btn-primary" :disabled="isCreatingTopic || !newTopicName.trim()">
            <span v-if="isCreatingTopic" class="spinner"></span>
            <span v-else>Crear</span>
          </button>
        </div>
      </div>
    </div>

    <!-- Modal Nueva Flashcard -->
    <div v-if="showNewCardModal" class="modal-overlay" @click="closeCardModal">
      <div class="modal" @click.stop>
        <h3>Nueva Flashcard</h3>
        <input
          v-model="newCard.question"
          type="text"
          placeholder="Pregunta"
          class="input"
          autofocus
        />
        <textarea
          v-model="newCard.answer"
          placeholder="Respuesta"
          class="textarea"
          rows="4"
        ></textarea>
        <div class="modal-actions">
          <button @click="closeCardModal" class="btn-secondary" :disabled="isCreatingCard">
            Cancelar
          </button>
          <button @click="createFlashcard" class="btn-primary" :disabled="isCreatingCard || !newCard.question.trim() || !newCard.answer.trim()">
            <span v-if="isCreatingCard" class="spinner"></span>
            <span v-else>Crear</span>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { db, auth } from './firebase'
import {
  collection,
  addDoc,
  getDocs,
  deleteDoc,
  doc,
  query,
  where,
  Timestamp
} from 'firebase/firestore'
import {
  createUserWithEmailAndPassword,
  signInWithEmailAndPassword,
  signOut,
  onAuthStateChanged
} from 'firebase/auth'

// Estados de autenticación
const user = ref(null)
const authMode = ref('login')
const authEmail = ref('')
const authPassword = ref('')
const authError = ref('')
const isAuthLoading = ref(false)

const topics = ref([])
const selectedTopic = ref(null)
const flashcards = ref([])
const showNewTopicModal = ref(false)
const showNewCardModal = ref(false)
const newTopicName = ref('')
const newCard = ref({ question: '', answer: '' })

// Estados de carga
const isLoadingTopics = ref(false)
const isLoadingFlashcards = ref(false)
const isCreatingTopic = ref(false)
const isCreatingCard = ref(false)
const isDeletingTopic = ref(null)
const isDeletingCard = ref(null)

// Autenticación
const register = async () => {
  authError.value = ''
  isAuthLoading.value = true
  try {
    await createUserWithEmailAndPassword(auth, authEmail.value, authPassword.value)
    authEmail.value = ''
    authPassword.value = ''
  } catch (error) {
    console.error('Error al registrarse:', error)
    if (error.code === 'auth/email-already-in-use') {
      authError.value = 'Este email ya está registrado'
    } else if (error.code === 'auth/weak-password') {
      authError.value = 'La contraseña debe tener al menos 6 caracteres'
    } else {
      authError.value = 'Error al registrarse. Intenta de nuevo.'
    }
  } finally {
    isAuthLoading.value = false
  }
}

const login = async () => {
  authError.value = ''
  isAuthLoading.value = true
  try {
    await signInWithEmailAndPassword(auth, authEmail.value, authPassword.value)
    authEmail.value = ''
    authPassword.value = ''
  } catch (error) {
    console.error('Error al iniciar sesión:', error)
    if (error.code === 'auth/user-not-found' || error.code === 'auth/wrong-password') {
      authError.value = 'Email o contraseña incorrectos'
    } else if (error.code === 'auth/invalid-credential') {
      authError.value = 'Email o contraseña incorrectos'
    } else {
      authError.value = 'Error al iniciar sesión. Intenta de nuevo.'
    }
  } finally {
    isAuthLoading.value = false
  }
}

const logout = async () => {
  try {
    await signOut(auth)
    topics.value = []
    flashcards.value = []
    selectedTopic.value = null
  } catch (error) {
    console.error('Error al cerrar sesión:', error)
  }
}

// Cargar temas
const loadTopics = async () => {
  if (!user.value) return
  
  isLoadingTopics.value = true
  try {
    const q = query(
      collection(db, 'topics'),
      where('userId', '==', user.value.uid)
    )
    const querySnapshot = await getDocs(q)
    topics.value = querySnapshot.docs.map(doc => ({
      id: doc.id,
      ...doc.data()
    }))
    
    // Contar flashcards por tema
    for (const topic of topics.value) {
      const flashcardsQuery = query(
        collection(db, 'flashcards'),
        where('topicId', '==', topic.id)
      )
      const flashcardsSnapshot = await getDocs(flashcardsQuery)
      topic.flashcardCount = flashcardsSnapshot.size
    }
  } catch (error) {
    console.error('Error al cargar temas:', error)
    alert('Error al cargar los temas')
  } finally {
    isLoadingTopics.value = false
  }
}

// Crear tema
const createTopic = async () => {
  if (!newTopicName.value.trim() || isCreatingTopic.value || !user.value) return

  isCreatingTopic.value = true
  try {
    await addDoc(collection(db, 'topics'), {
      name: newTopicName.value.trim(),
      userId: user.value.uid,
      createdAt: Timestamp.now()
    })
    newTopicName.value = ''
    showNewTopicModal.value = false
    await loadTopics()
  } catch (error) {
    console.error('Error al crear tema:', error)
    alert('Error al crear el tema')
  } finally {
    isCreatingTopic.value = false
  }
}

// Eliminar tema
const deleteTopic = async (topicId) => {
  if (!confirm('¿Eliminar este tema y todas sus flashcards?')) return
  if (isDeletingTopic.value) return

  isDeletingTopic.value = topicId
  try {
    // Eliminar flashcards del tema
    const flashcardsQuery = query(
      collection(db, 'flashcards'),
      where('topicId', '==', topicId)
    )
    const flashcardsSnapshot = await getDocs(flashcardsQuery)
    const deletePromises = flashcardsSnapshot.docs.map(doc =>
      deleteDoc(doc.ref)
    )
    await Promise.all(deletePromises)

    // Eliminar tema
    await deleteDoc(doc(db, 'topics', topicId))
    await loadTopics()
  } catch (error) {
    console.error('Error al eliminar tema:', error)
    alert('Error al eliminar el tema')
  } finally {
    isDeletingTopic.value = null
  }
}

// Seleccionar tema
const selectTopic = async (topic) => {
  selectedTopic.value = topic
  await loadFlashcards(topic.id)
}

// Cargar flashcards
const loadFlashcards = async (topicId) => {
  isLoadingFlashcards.value = true
  try {
    const q = query(
      collection(db, 'flashcards'),
      where('topicId', '==', topicId)
    )
    const querySnapshot = await getDocs(q)
    flashcards.value = querySnapshot.docs.map(doc => ({
      id: doc.id,
      ...doc.data(),
      isFlipped: false
    })).sort((a, b) => b.createdAt?.seconds - a.createdAt?.seconds)
  } catch (error) {
    console.error('Error al cargar flashcards:', error)
    alert('Error al cargar las flashcards')
  } finally {
    isLoadingFlashcards.value = false
  }
}

// Crear flashcard
const createFlashcard = async () => {
  if (!newCard.value.question.trim() || !newCard.value.answer.trim() || isCreatingCard.value || !user.value) return

  isCreatingCard.value = true
  try {
    await addDoc(collection(db, 'flashcards'), {
      topicId: selectedTopic.value.id,
      userId: user.value.uid,
      question: newCard.value.question.trim(),
      answer: newCard.value.answer.trim(),
      createdAt: Timestamp.now()
    })
    newCard.value = { question: '', answer: '' }
    showNewCardModal.value = false
    await loadFlashcards(selectedTopic.value.id)
  } catch (error) {
    console.error('Error al crear flashcard:', error)
    alert('Error al crear la flashcard')
  } finally {
    isCreatingCard.value = false
  }
}

// Eliminar flashcard
const deleteFlashcard = async (cardId) => {
  if (!confirm('¿Eliminar esta flashcard?')) return
  if (isDeletingCard.value) return

  isDeletingCard.value = cardId
  try {
    await deleteDoc(doc(db, 'flashcards', cardId))
    await loadFlashcards(selectedTopic.value.id)
  } catch (error) {
    console.error('Error al eliminar flashcard:', error)
    alert('Error al eliminar la flashcard')
  } finally {
    isDeletingCard.value = null
  }
}

// Voltear flashcard
const toggleFlip = (card) => {
  card.isFlipped = !card.isFlipped
}

// Cerrar modales
const closeTopicModal = () => {
  showNewTopicModal.value = false
  newTopicName.value = ''
}

const closeCardModal = () => {
  showNewCardModal.value = false
  newCard.value = { question: '', answer: '' }
}

// Observar cambios de autenticación
onMounted(() => {
  onAuthStateChanged(auth, (currentUser) => {
    user.value = currentUser
    if (currentUser) {
      loadTopics()
    }
  })
})
</script>

<style scoped>
.app {
  min-height: 100vh;
}

/* Auth View */
.auth-view {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2rem;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.auth-container {
  background: var(--bg);
  border-radius: var(--radius);
  padding: 3rem;
  width: 100%;
  max-width: 420px;
  box-shadow: 0 20px 25px -5px rgb(0 0 0 / 0.1);
}

.auth-logo {
  font-size: 2.5rem;
  text-align: center;
  margin-bottom: 0.5rem;
}

.auth-subtitle {
  text-align: center;
  color: var(--text-secondary);
  margin-bottom: 2rem;
}

.auth-tabs {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 2rem;
  border-bottom: 2px solid var(--border);
}

.auth-tab {
  flex: 1;
  padding: 0.75rem;
  background: transparent;
  color: var(--text-secondary);
  border: none;
  border-bottom: 2px solid transparent;
  margin-bottom: -2px;
  font-weight: 500;
  transition: all 0.2s;
}

.auth-tab.active {
  color: var(--primary);
  border-bottom-color: var(--primary);
}

.auth-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.btn-block {
  width: 100%;
  padding: 1rem;
  font-size: 1rem;
}

.error-message {
  color: #ef4444;
  text-align: center;
  font-size: 0.9rem;
  margin: 0;
}

.header {
  background: var(--bg);
  border-bottom: 1px solid var(--border);
  padding: 1.5rem 0;
  margin-bottom: 2rem;
}

.header-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.user-section {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.user-email {
  color: var(--text-secondary);
  font-size: 0.9rem;
}

.btn-logout {
  background: var(--bg-secondary);
  color: var(--text);
  padding: 0.5rem 1rem;
  border-radius: 8px;
  font-size: 0.9rem;
  font-weight: 500;
}

.btn-logout:hover {
  background: var(--border);
}

.logo {
  font-size: 1.5rem;
  font-weight: 600;
}

.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 1.5rem;
}

.view-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
  gap: 1rem;
  flex-wrap: wrap;
}

.view-header h2 {
  font-size: 1.75rem;
  font-weight: 600;
}

.btn-primary {
  background: var(--primary);
  color: white;
  padding: 0.75rem 1.5rem;
  border-radius: var(--radius);
  font-weight: 500;
  font-size: 0.95rem;
}

.btn-primary:hover:not(:disabled) {
  background: var(--primary-hover);
}

.btn-secondary {
  background: var(--bg-secondary);
  color: var(--text);
  padding: 0.75rem 1.5rem;
  border-radius: var(--radius);
  font-weight: 500;
}

.btn-secondary:hover:not(:disabled) {
  background: var(--border);
}

.btn-back {
  background: transparent;
  color: var(--text-secondary);
  padding: 0.5rem 1rem;
  border-radius: var(--radius);
  font-weight: 500;
}

.btn-back:hover:not(:disabled) {
  background: var(--bg-secondary);
  color: var(--text);
}

.btn-icon {
  background: transparent;
  font-size: 1.2rem;
  padding: 0.25rem 0.5rem;
  border-radius: 6px;
}

.btn-icon:hover:not(:disabled) {
  background: var(--bg-secondary);
}

.empty-state {
  text-align: center;
  padding: 4rem 2rem;
  color: var(--text-secondary);
}

.topics-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 1.5rem;
}

.topic-card {
  background: var(--bg);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 1.5rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.topic-card:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow-lg);
  border-color: var(--primary);
}

.topic-header {
  display: flex;
  justify-content: space-between;
  align-items: start;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.topic-header h3 {
  font-size: 1.25rem;
  font-weight: 600;
  flex: 1;
}

.topic-count {
  color: var(--text-secondary);
  font-size: 0.9rem;
}

.flashcards-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 1.5rem;
}

.flashcard {
  height: 250px;
  perspective: 1000px;
  cursor: pointer;
}

.flashcard-inner {
  position: relative;
  width: 100%;
  height: 100%;
  transition: transform 0.6s;
  transform-style: preserve-3d;
}

.flashcard.flipped .flashcard-inner {
  transform: rotateY(180deg);
}

.flashcard-front,
.flashcard-back {
  position: absolute;
  width: 100%;
  height: 100%;
  backface-visibility: hidden;
  background: var(--bg);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 2rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
}

.flashcard-front {
  gap: 1rem;
}

.flashcard-front p {
  font-size: 1.1rem;
  font-weight: 500;
  line-height: 1.5;
}

.flip-hint {
  color: var(--text-secondary);
  font-size: 0.85rem;
}

.flashcard-back {
  transform: rotateY(180deg);
  gap: 1.5rem;
}

.flashcard-back p {
  font-size: 1rem;
  line-height: 1.6;
  color: var(--text-secondary);
}

.btn-delete {
  background: #ef4444;
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 8px;
  font-size: 0.9rem;
}

.btn-delete:hover:not(:disabled) {
  background: #dc2626;
}

.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1rem;
  z-index: 1000;
}

.modal {
  background: var(--bg);
  border-radius: var(--radius);
  padding: 2rem;
  width: 100%;
  max-width: 500px;
  box-shadow: var(--shadow-lg);
}

.modal h3 {
  font-size: 1.5rem;
  margin-bottom: 1.5rem;
}

.input,
.textarea {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid var(--border);
  border-radius: 8px;
  font-size: 1rem;
  margin-bottom: 1rem;
  background: var(--bg);
  color: var(--text);
}

.input:focus,
.textarea:focus {
  outline: none;
  border-color: var(--primary);
}

.textarea {
  resize: vertical;
}

.modal-actions {
  display: flex;
  gap: 1rem;
  justify-content: flex-end;
}

/* Skeleton Loading */
.skeleton-card {
  background: var(--bg);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 1.5rem;
  animation: pulse 1.5s ease-in-out infinite;
}

.skeleton-flashcard {
  height: 250px;
}

@keyframes pulse {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}

/* Spinner */
.spinner {
  display: inline-block;
  width: 14px;
  height: 14px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top-color: white;
  border-radius: 50%;
  animation: spin 0.6s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

/* Disabled buttons */
button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

button:disabled:active {
  transform: none;
}

@media (max-width: 640px) {
  .view-header {
    flex-direction: column;
    align-items: stretch;
  }

  .topics-grid,
  .flashcards-grid {
    grid-template-columns: 1fr;
  }

  .auth-container {
    padding: 2rem;
  }

  .header-content {
    flex-direction: column;
    gap: 1rem;
    align-items: stretch;
  }

  .user-section {
    flex-direction: column;
    align-items: stretch;
  }
}
</style>
