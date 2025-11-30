<script setup lang="ts">
import PouchDB from 'pouchdb'
import { onMounted, ref } from 'vue';
import findPlugin from "pouchdb-find";
PouchDB.plugin(findPlugin);

// ===== INTERFACES =====
declare interface Comment {
  _id: string
  content: string
  author: string
  creation_date: string
}

declare interface Post {
  _id: string
  _rev?: string
  type: 'post'
  title: string
  content: string
  likes: number
  comments: Comment[]
  creation_date: any
  updated_date: any
}

const storage = ref();
const url = 'http://inoe.wenger:IWtramp54HEIG/@localhost:5984/infradon_inoe_db/'
const opts = { live: true, retry: true };
const postsData = ref<Post[]>([])
const sync = ref();

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase()
});

const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const localdb = new PouchDB('local')
  if (localdb) {
    console.log("Connected to collection : " + localdb?.name)
    storage.value = localdb

    // Créer les indexes
    createIndexes()

    // Répliquer PUIS chercher les données
    localdb.replicate.from(url)
    .on('complete', () => {
      console.log('Réplication complète')
      fetchData()  // ← Une seule fois après réplication
      syncData()   // ← Puis démarrer la sync continue
    })
  } else {
    console.warn('Something went wrong')
  }
}

const createIndexes = () => {
  console.log('=> Création des indexes');

  // Index sur le titre (pour la recherche)
  storage.value.createIndex({
    index: {
      fields: ['title']
    }
  })
  .then(() => console.log("Index 'title' créé"))
  .catch((err: any) => console.error("Erreur index title:", err))

  // Index sur le contenu (pour la recherche)
  storage.value.createIndex({
    index: {
      fields: ['content']
    }
  })
  .then(() => console.log("Index 'content' créé"))
  .catch((err: any) => console.error("Erreur index content:", err))

  // Index sur le nombre de likes (pour le tri)
  storage.value.createIndex({
    index: {
      fields: ['likes']
    }
  })
  .then(() => console.log("Index 'likes' créé"))
  .catch((err: any) => console.error("Erreur index likes:", err))

  // Index sur le type (pour filtrer uniquement les posts)
  storage.value.createIndex({
    index: {
      fields: ['type']
    }
  })
  .then(() => console.log("Index 'type' créé"))
  .catch((err: any) => console.error("Erreur index type:", err))
}

const syncData = () => {
  sync.value = storage.value
  .sync(url, opts)
  .on('change', fetchData)
}

const toggle = () => {
  if (sync.value) {
    sync.value.cancel()
    sync.value = null
  } else {
    syncData()
  }
}

const search = (event: any) => {
  event.target.blur()

  if (event.target.value === "") {
    fetchData();
  } else {
    storage.value.find({
      selector: {content: `Contenu du document : ${event.target.value}`}
    })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.docs)
      postsData.value = result.docs
      // console.log(postsData)
    })
    .catch((error: any) => {
      console.error('Erreur lors de la récupération des données :', error)
    })
  }
}

const searchReset = () => {
  const searchInput = document.querySelector(".search") as HTMLInputElement
  if (searchInput) {
    searchInput.value = ""
  }
  fetchData()
}

const fetchData = (): any => {
  storage.value
    .find({
      selector: { type: 'post' }
    })
    .then((result: any) => {
      console.log('=> Posts récupérés :', result.docs.length)
      postsData.value = result.docs as Post[]
    })
    .catch((error: any) => {
      console.error('Erreur lors de la récupération des données :', error)
    })
}

const createDoc = (): any => {
  // Récupérer les valeurs des inputs
  const titleInput = document.querySelector(".input-title") as HTMLInputElement
  const contentInput = document.querySelector(".input-content") as HTMLInputElement

  const title = titleInput?.value.trim()
  const content = contentInput?.value.trim()

  // Validation
  if (!title || !content) {
    console.warn('Titre et contenu sont obligatoires')
    return
  }

  console.log('=> Création d\'un nouveau post');

  const newPost: Post = {
    _id: `post_${Date.now()}`,
    type: 'post',
    title: title,
    content: content,
    likes: 0,
    comments: [],
    creation_date: new Date().toISOString(),
    updated_date: new Date().toISOString()
  }

  storage.value
    .post(newPost)
    .then((response: any) => {
      console.log('Post créé :', response)
      // Vider les inputs
      titleInput.value = ""
      contentInput.value = ""
      fetchData()
    })
    .catch((err: any) => {
      console.error('Erreur création post :', err)
    })
}

const deleteDoc = (post: Post): any => {
  console.log('=> Suppression du post:', post._id);

  storage.value
    .remove(post)
    .then((response: any) => {
      console.log('Post supprimé :', response)
      fetchData()
    })
    .catch((err: any) => {
      console.error('Erreur suppression post :', err)
    })
}

const updateDoc = (post: Post): any => {
  // Récupérer les nouvelles valeurs
  const newTitle = prompt('Nouveau titre:', post.title)
  if (newTitle === null) return  // Annulation

  const newContent = prompt('Nouveau contenu:', post.content)
  if (newContent === null) return  // Annulation

  // Validation
  if (!newTitle.trim() || !newContent.trim()) {
    console.warn('Titre et contenu sont obligatoires')
    return
  }

  console.log('=> Modification du post:', post._id);

  // Modifier le post
  post.title = newTitle.trim()
  post.content = newContent.trim()
  post.updated_date = new Date().toISOString()

  storage.value
    .put(post)
    .then((response: any) => {
      console.log('Post modifié :', response)
      fetchData()
    })
    .catch((err: any) => {
      console.error('Erreur modification post :', err)
    })
}

// ===== SYSTÈME DE LIKES =====
const toggleLike = (post: Post): any => {
  console.log('=> Toggle like sur post:', post._id);

  // Incrémenter le compteur
  post.likes++
  post.updated_date = new Date().toISOString()

  storage.value
    .put(post)
    .then((response: any) => {
      console.log('✓ Post liké :', response)
      fetchData()
    })
    .catch((err: any) => {
      console.error('Erreur modification like :', err)
    })
}

// ===== SYSTÈME DE COMMENTAIRES =====
const addComment = (post: Post): any => {
  // Récupérer le contenu du commentaire depuis l'input
  const commentInput = document.querySelector(`.comment-input-${post._id}`) as HTMLInputElement
  const commentContent = commentInput?.value.trim()

  // Validation
  if (!commentContent) {
    console.warn('Le commentaire ne peut pas être vide')
    return
  }

  console.log('=> Ajout d\'un commentaire au post:', post._id);

  // Créer le nouveau commentaire
  const newComment: Comment = {
    _id: `comment_${Date.now()}`,
    content: commentContent,
    author: 'Toi',
    creation_date: new Date().toISOString()
  }

  // Ajouter le commentaire au post
  post.comments.push(newComment)
  post.updated_date = new Date().toISOString()

  // Sauvegarder le post modifié
  storage.value
    .put(post)
    .then((response: any) => {
      console.log('Commentaire ajouté :', response)
      commentInput.value = ""  // Vider l'input
      fetchData()
    })
    .catch((err: any) => {
      console.error('Erreur ajout commentaire :', err)
    })
}

// ===== FACTORY - GÉNÉRER DONNÉES TEST =====
const generateTestData = async () => {
  console.log('=> Génération des données de test...');

  const titles = [
    'Mon premier post',
    'Découverte intéressante',
    'Réflexion du jour',
    'News importante',
    'Question pour vous',
    'Partage d\'expérience',
    'Conseil utile',
    'Actualité tech'
  ]

  const contents = [
    'Ceci est le contenu du post numéro',
    'Je voudrais partager avec vous',
    'Vous pensez quoi de',
    'Récemment j\'ai découvert',
    'Pourquoi ne pas essayer',
    'Voici mon avis sur',
    'Important à savoir',
    'Fait intéressant'
  ]

  for (let i = 0; i < 15; i++) {
    const post: Post = {
      _id: `post_test_${i}_${Date.now()}`,
      type: 'post',
      title: titles[i % titles.length] + ' ' + i,
      content: contents[i % contents.length] + ' ' + i,
      likes: Math.floor(Math.random() * 50),
      comments: [],
      creation_date: new Date(Date.now() - Math.random() * 7 * 24 * 60 * 60 * 1000).toISOString(),
      updated_date: new Date().toISOString()
    }

    try {
      await storage.value.post(post)
      console.log('Post ' + i + ' créé')
    } catch (err: any) {
      console.error('Erreur création post test:', err)
    }
  }

  console.log('=> Génération terminée');
  fetchData()
}

// ===== SUPPRIMER TOUS LES POSTS =====
const deleteAllPosts = async () => {
  if (!confirm('Êtes-vous sûr ? Tous les posts seront supprimés !')) {
    return;
  }

  console.log('=> Suppression définitive de tous les posts...');

  try {
    // Supprimer la base de données locale COMPLÈTEMENT
    await storage.value.destroy()

    // Recréer une base vide
    storage.value = new PouchDB('local')

    // Recréer les indexes
    createIndexes()

    console.log('Tous les posts supprimés');
    postsData.value = []
  } catch (err: any) {
    console.error('Erreur suppression posts:', err)
  }
}

</script>

<template>
  <h1>Fetch Data</h1>
  <div>
    <label v-if="sync"> Online</label>
    <label v-else> Offline</label>
    <input @click="toggle" type="checkbox" name="toggleSync" />
    <label for="toggleSync">Toggle Sync</label>
  </div>
  <div>
    <input type="text" placeholder="Search" @keyup.enter="search" class="search">
    <button @click="searchReset">X</button>
  </div>

  <!-- SECTION CRÉATION POST -->
  <div class="create-section">
    <h2>📝 Créer un post</h2>
    <input
      type="text"
      class="input-title"
      placeholder="Titre du post"
    >
    <textarea
      class="input-content"
      placeholder="Contenu du post"
    ></textarea>
    <button @click="createDoc" class="btn-primary">➕ Ajouter un post</button>
    <button @click="generateTestData" class="btn-secondary">🧪 Générer données test (15 posts)</button>
    <button @click="deleteAllPosts" class="btn-danger">🗑️ Supprimer tous les posts</button>
  </div>

  <!-- LISTE DES POSTS -->
  <article v-for="post in postsData" v-bind:key="(post as any)._id" class="post-card">
    <h2>{{ post.title }}</h2>
    <p>{{ post.content }}</p>

    <!-- BOUTONS D'ACTIONS POST -->
    <div class="post-actions">
      <button @click="updateDoc(post)" class="btn-small">✏️ Modifier</button>
      <button @click="deleteDoc(post)" class="btn-small-danger">🗑️ Supprimer</button>
    </div>

    <!-- BOUTON LIKES -->
    <button @click="toggleLike(post)" class="btn-like">
      👍 {{ post.likes }} likes
    </button>

    <!-- SECTION COMMENTAIRES -->
    <div class="comments-section">
      <h3>💬 Commentaires ({{ post.comments.length }})</h3>
      <div v-for="comment in post.comments" :key="comment._id" class="comment">
        <strong>{{ comment.author }}</strong>
        <span class="comment-date">({{ new Date(comment.creation_date).toLocaleString() }})</span>
        <p>{{ comment.content }}</p>
      </div>
      <div class="comment-input-wrapper">
        <input
          type="text"
          :class="`comment-input-${post._id}`"
          placeholder="Ajouter un commentaire"
          class="comment-input"
        >
        <button @click="addComment(post)" class="btn-comment">
          💬 Commenter
        </button>
      </div>
    </div>
  </article>
</template>

<style scoped>
/* ===== VARIABLES COULEURS ===== */
:root {
  --primary: #3498db;
  --primary-dark: #2980b9;
  --success: #27ae60;
  --warning: #f39c12;
  --danger: #e74c3c;
  --secondary: #9b59b6;
  --light-bg: #ecf0f1;
  --border-color: #bdc3c7;
  --text-muted: #95a5a6;
}

/* ===== GÉNÉRIQUES ===== */
body {
  font-family: Arial, sans-serif;
  background-color: #f9f9f9;
  padding: 20px;
}

h1 {
  text-align: center;
  color: #333;
}

h2 {
  color: #333;
  margin-top: 0;
}

h3 {
  color: #333;
  font-size: 1em;
}

p {
  color: #555;
}

/* ===== INPUTS ===== */
input[type="text"],
textarea {
  width: 100%;
  padding: 8px;
  margin-bottom: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-sizing: border-box;
  font-family: Arial, sans-serif;
}

textarea {
  resize: vertical;
  min-height: 60px;
}

input[type="checkbox"] {
  margin-right: 5px;
}

/* ===== BOUTONS ===== */
button {
  padding: 8px 12px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  background-color: #007bff;
  color: white;
  margin-right: 5px;
  margin-bottom: 8px;
}

button:hover {
  background-color: #0056b3;
}

.btn-danger {
  background-color: #dc3545;
}

.btn-danger:hover {
  background-color: #c82333;
}

/* ===== SECTIONS ===== */
.create-section {
  background-color: #f0f0f0;
  padding: 15px;
  border-radius: 4px;
  margin-bottom: 15px;
}

.post-card {
  background: white;
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 15px;
  margin-bottom: 15px;
}

.post-actions {
  margin: 10px 0;
  padding-bottom: 10px;
}

/* ===== COMMENTAIRES ===== */
.comments-section {
  background-color: #f9f9f9;
  padding: 12px;
  margin-top: 12px;
  border-radius: 4px;
  border: 1px solid #eee;
}

.comment {
  background-color: white;
  padding: 10px;
  margin-bottom: 8px;
  border-left: 3px solid #007bff;
  border-radius: 3px;
}

.comment strong {
  color: #333;
}

.comment-date {
  font-size: 0.85em;
  color: #999;
  margin-left: 5px;
}

.comment p {
  margin: 5px 0 0 0;
  color: #555;
}

.comment-input-wrapper {
  display: flex;
  gap: 8px;
  margin-top: 8px;
}

.comment-input {
  flex: 1;
  margin-bottom: 0 !important;
}
</style>
