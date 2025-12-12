<script setup lang="ts">
import PouchDB from 'pouchdb'
import { onMounted, ref } from 'vue'
import findPlugin from 'pouchdb-find'
PouchDB.plugin(findPlugin)

// ===== INTERFACES =====
declare interface Comment {
  _id: string
  _rev?: string
  postId: string
  content: string
  author: string
  type: 'comment'
  creation_date: string
}

declare interface Post {
  _id: string
  _rev?: string
  type: 'post'
  title: string
  content: string
  likes: number
  comments?: Comment[]
  creation_date: string
  updated_date: string
}

// ===== VARIABLES =====
const postsDB = ref()
const commentsDB = ref()
const postsUrl = 'http://inoe.wenger:IWtramp54HEIG/@localhost:5984/infradon_inoe_posts/'
const commentsUrl = 'http://inoe.wenger:IWtramp54HEIG/@localhost:5984/infradon_inoe_comments/'
const opts = { live: true, retry: true }
const postsData = ref<Post[]>([])
const postsSync = ref()
const commentsSync = ref()
const replicationStatus = ref<string>('En attente...')
const isReplicating = ref<boolean>(false)
const isOnline = ref<boolean>(true)
const syncStatus = ref<string>('Synchronisé')
const lastSyncTime = ref<string>('')
const offlineMode = ref<boolean>(false)
const offlineChanges = ref<number>(0)

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
})

const initDatabase = async () => {
  console.log('=> Connexion aux bases de données')
  const localPostsDB = new PouchDB('local_posts')
  const localCommentsDB = new PouchDB('local_comments')

  if (localPostsDB && localCommentsDB) {
    console.log('Connected to posts collection : ' + localPostsDB?.name)
    console.log('Connected to comments collection : ' + localCommentsDB?.name)
    postsDB.value = localPostsDB
    commentsDB.value = localCommentsDB

    await createIndexes()
    replicateFromServer()
  } else {
    console.warn('Something went wrong')
  }
}

const replicateFromServer = () => {
  console.log('=> Début de la réplication depuis le serveur')
  isReplicating.value = true
  replicationStatus.value = '⏳ Réplication en cours...'

  let postsReplicationDone = false
  let commentsReplicationDone = false

  const checkBothComplete = () => {
    if (postsReplicationDone && commentsReplicationDone) {
      console.log('Réplication des 2 bases complète')
      isReplicating.value = false
      replicationStatus.value = '✅ Synchronisation complète (posts + commentaires)'
      fetchData()
      syncData()
    }
  }

  // Réplication des posts
  postsDB.value.replicate
    .from(postsUrl)
    .on('change', (info: any) => {
      console.log('=> Posts : ' + info.docs_read + ' documents lus')
      replicationStatus.value = `⏳ Posts: ${info.docs_read} documents...`
    })
    .on('complete', (info: any) => {
      console.log('Réplication posts complète :', info.docs_read + ' documents')
      postsReplicationDone = true
      checkBothComplete()
    })
    .on('error', (err: any) => {
      console.error('Erreur réplication posts :', err)
      isReplicating.value = false
      replicationStatus.value = 'Erreur posts : ' + err.message
    })

  // Réplication des commentaires
  commentsDB.value.replicate
    .from(commentsUrl)
    .on('change', (info: any) => {
      console.log('=> Commentaires : ' + info.docs_read + ' documents lus')
    })
    .on('complete', (info: any) => {
      console.log('Réplication commentaires complète :', info.docs_read + ' documents')
      commentsReplicationDone = true
      checkBothComplete()
    })
    .on('error', (err: any) => {
      console.error('Erreur réplication commentaires :', err)
      isReplicating.value = false
      replicationStatus.value = 'Erreur commentaires : ' + err.message
    })
}

const syncData = () => {
  if (offlineMode.value) {
    console.log('=> Pas de sync (mode hors ligne activé)')
    return
  }

  console.log('=> Lancement de la synchronisation bidirectionnelle (2 bases)')

  // Synchronisation des posts
  postsSync.value = postsDB.value
    .sync(postsUrl, opts)
    .on('change', (change: any) => {
      console.log('=> Posts sync:', change.direction, change.change.docs.length + ' doc(s)')
      lastSyncTime.value = new Date().toLocaleTimeString()
    })
    .on('paused', async () => {
      console.log('=> Posts sync en pause')
      await resolveConflicts()
      syncStatus.value = 'Synchronisé'
      isOnline.value = true
    })
    .on('active', () => {
      console.log('=> Posts sync active')
      syncStatus.value = '🔄 Synchronisation...'
      isOnline.value = true
    })
    .on('error', (err: any) => {
      console.error('Erreur sync posts :', err)
      syncStatus.value = 'Erreur posts : ' + err.message
      isOnline.value = false
    })

  // Synchronisation des commentaires
  commentsSync.value = commentsDB.value
    .sync(commentsUrl, opts)
    .on('change', (change: any) => {
      console.log('=> Commentaires sync:', change.direction, change.change.docs.length + ' doc(s)')
      lastSyncTime.value = new Date().toLocaleTimeString()
    })
    .on('paused', () => {
      console.log('=> Commentaires sync en pause')
    })
    .on('active', () => {
      console.log('=> Commentaires sync active')
    })
    .on('error', (err: any) => {
      console.error('Erreur sync commentaires :', err)
      syncStatus.value = 'Erreur commentaires : ' + err.message
      isOnline.value = false
    })
}

const search = (event: any) => {
  event.target.blur()

  const query = event.target.value.trim()

  if (query === '') {
    fetchData()
    return
  }

  console.log('=> Recherche sur :', query)

  postsDB.value
    .find({
      selector: {
        $or: [{ title: { $regex: query } }, { content: { $regex: query } }],
      },
    })
    .then(async (result: any) => {
      console.log('=> Résultats trouvés :', result.docs.length)
      const postsWithComments = await attachComments(result.docs)
      postsData.value = postsWithComments as Post[]
    })
    .catch((error: any) => {
      console.error('Erreur lors de la recherche :', error)
    })
}

const createIndexes = async () => {
  console.log('=> Création des indexes')

  try {
    // Index pour la base posts
    await postsDB.value.createIndex({ index: { fields: ['title'] } })
    console.log("Index posts 'title' créé")

    await postsDB.value.createIndex({ index: { fields: ['content'] } })
    console.log("Index posts 'content' créé")

    await postsDB.value.createIndex({ index: { fields: ['likes'] } })
    console.log("Index posts 'likes' créé")

    // Index pour la base commentaires
    await commentsDB.value.createIndex({ index: { fields: ['postId'] } })
    console.log("Index comments 'postId' créé")

    await commentsDB.value.createIndex({ index: { fields: ['content'] } })
    console.log("Index comments 'content' créé")
  } catch (err: any) {
    console.error('Erreur création indexes:', err)
  }
}

const sortByLikes = (): any => {
  console.log('=> Tri par nombre de likes')

  postsDB.value
    .find({
      selector: {
        likes: { $gte: 0 },
      },
      sort: [{ likes: 'desc' }],
    })
    .then(async (result: any) => {
      console.log('=> Posts triés par likes :', result.docs.length)
      const postsWithComments = await attachComments(result.docs)
      postsData.value = postsWithComments as Post[]
    })
    .catch((error: any) => {
      console.error('Erreur lors du tri :', error)
    })
}

const fetchData = async (posts?: Post[]): Promise<any> => {
  // Si pas de posts en paramètre, récupérer les posts de la base
  if (!posts) {
    postsDB.value
      .allDocs({ include_docs: true })
      .then(async (result: any) => {
        // Filtrer les documents système (_design, _local)
        const validRows = result.rows.filter((row: any) => !row.id.startsWith('_'))
        console.log('=> Posts récupérés :', validRows.length)
        posts = validRows.map((row: any) => row.doc) as Post[]

        // Ajouter les commentaires
        const postsWithComments = await attachComments(posts)
        postsData.value = postsWithComments as Post[]
      })
      .catch((error: any) => {
        console.error('Erreur lors de la récupération des données :', error)
      })
  }
}

// Fonction utilitaire pour ajouter les commentaires aux posts
const attachComments = async (posts: Post[]): Promise<Post[]> => {
  const results = await Promise.all(
    posts.map((post) =>
      commentsDB.value
        .find({
          selector: {
            postId: post._id,
          },
        })
        .then((commentsResult: any) => ({
          ...post,
          comments: commentsResult.docs as Comment[],
        })),
    ),
  )
  return results
}

const createDoc = (): any => {
  // Récupérer les valeurs des inputs
  const titleInput = document.querySelector('.input-title') as HTMLInputElement
  const contentInput = document.querySelector('.input-content') as HTMLInputElement

  const title = titleInput?.value.trim()
  const content = contentInput?.value.trim()

  // Validation
  if (!title || !content) {
    console.warn('Titre et contenu sont obligatoires')
    return
  }

  console.log("=> Création d'un nouveau post")

  const newPost: Post = {
    _id: `post_${Date.now()}`,
    type: 'post',
    title: title,
    content: content,
    likes: 0,
    creation_date: new Date().toISOString(),
    updated_date: new Date().toISOString(),
  }

  postsDB.value
    .post(newPost)
    .then((response: any) => {
      console.log('Post créé :', response)
      titleInput.value = ''
      contentInput.value = ''
      trackLocalChange()
      fetchData()
    })
    .catch((err: any) => {
      console.error('Erreur création post :', err)
    })
}

const deleteDoc = (post: Post): any => {
  console.log('=> Suppression du post:', post._id)

  postsDB.value
    .remove(post)
    .then((response: any) => {
      console.log('Post supprimé :', response)
      trackLocalChange()
      fetchData()
    })
    .catch((err: any) => {
      console.error('Erreur suppression post :', err)
    })
}

const updateDoc = (post: Post): any => {
  // Récupérer les nouvelles valeurs
  const newTitle = prompt('Nouveau titre:', post.title)
  if (newTitle === null) return

  const newContent = prompt('Nouveau contenu:', post.content)
  if (newContent === null) return

  // Validation
  if (!newTitle.trim() || !newContent.trim()) {
    console.warn('Titre et contenu sont obligatoires')
    return
  }

  console.log('=> Modification du post:', post._id)

  // Modifier le post
  post.title = newTitle.trim()
  post.content = newContent.trim()
  post.updated_date = new Date().toISOString()

  postsDB.value
    .put(post)
    .then((response: any) => {
      console.log('Post modifié :', response)
      trackLocalChange()
      fetchData()
    })
    .catch((err: any) => {
      console.error('Erreur modification post :', err)
    })
}

// ===== SYSTÈME DE LIKES =====
const toggleLike = (post: Post): any => {
  console.log('=> Toggle like sur post:', post._id)

  // Incrémenter le compteur
  post.likes++
  post.updated_date = new Date().toISOString()

  postsDB.value
    .put(post)
    .then((response: any) => {
      console.log('Post liké :', response)
      trackLocalChange()
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

  console.log("=> Ajout d'un commentaire au post:", post._id)

  // Créer le nouveau commentaire
  const newComment: Comment = {
    _id: `comment_${Date.now()}`,
    postId: post._id,
    content: commentContent,
    author: 'Toi',
    type: 'comment',
    creation_date: new Date().toISOString(),
  }

  // Sauvegarder le commentaire
  commentsDB.value
    .post(newComment)
    .then((response: any) => {
      console.log('Commentaire ajouté :', response)
      commentInput.value = ''
      trackLocalChange()
      fetchData()
    })
    .catch((err: any) => {
      console.error('Erreur ajout commentaire :', err)
    })
}

const deleteComment = (post: Post, comment: Comment): any => {
  console.log('=> Suppression du commentaire:', comment._id)

  // Supprimer le commentaire
  commentsDB.value
    .remove(comment)
    .then((response: any) => {
      console.log('Commentaire supprimé :', response)
      trackLocalChange()
      fetchData()
    })
    .catch((err: any) => {
      console.error('Erreur suppression commentaire :', err)
    })
}

const updateComment = (post: Post, comment: Comment): any => {
  console.log('=> Modification du commentaire:', comment._id)

  // Récupérer le nouveau contenu
  const newContent = prompt('Nouveau contenu du commentaire:', comment.content)
  if (newContent === null) return

  // Validation
  if (!newContent.trim()) {
    console.warn('Le commentaire ne peut pas être vide')
    return
  }

  // Modifier le commentaire
  comment.content = newContent.trim()

  commentsDB.value
    .put(comment)
    .then((response: any) => {
      console.log('Commentaire modifié :', response)
      trackLocalChange()
      fetchData()
    })
    .catch((err: any) => {
      console.error('Erreur modification commentaire :', err)
    })
}

// ===== FACTORY - GÉNÉRER DONNÉES TEST =====
const generateTestData = async () => {
  console.log('=> Génération des données de test...')

  const titles = [
    'Mon premier post',
    'Découverte intéressante',
    'Réflexion du jour',
    'News importante',
    'Question pour vous',
    "Partage d'expérience",
    'Conseil utile',
    'Actualité tech',
  ]

  const contents = [
    'Ceci est le contenu du post numéro',
    'Je voudrais partager avec vous',
    'Vous pensez quoi de',
    "Récemment j'ai découvert",
    'Pourquoi ne pas essayer',
    'Voici mon avis sur',
    'Important à savoir',
    'Fait intéressant',
  ]

  const commentTexts = [
    'Super post !',
    'Très intéressant',
    "Je suis d'accord",
    'À découvrir absolument',
    'Merci pour le partage',
    'Excellent conseil',
    "C'est vrai !",
    'Bien dit',
  ]

  // ===== ÉTAPE 1 : CRÉER ET SAUVEGARDER LES POSTS =====
  const postIds: string[] = []

  for (let i = 0; i < 15; i++) {
    const postId = `post_test_${i}_${Date.now()}`
    postIds.push(postId)

    const post: Post = {
      _id: postId,
      type: 'post',
      title: titles[i % titles.length] + ' ' + i,
      content: contents[i % contents.length] + ' ' + i,
      likes: Math.floor(Math.random() * 50),
      creation_date: new Date(Date.now() - Math.random() * 7 * 24 * 60 * 60 * 1000).toISOString(),
      updated_date: new Date().toISOString(),
    }

    try {
      await postsDB.value.post(post)
      console.log('Post ' + i + ' créé')
    } catch (err: any) {
      console.error('Erreur création post test:', err)
    }
  }

  console.log('=> ' + postIds.length + ' posts créés')

  // ===== ÉTAPE 2 : CRÉER ET SAUVEGARDER LES COMMENTAIRES INDÉPENDANTS =====
  let totalComments = 0

  for (let i = 0; i < postIds.length; i++) {
    const nbComments = Math.floor(Math.random() * 5)

    for (let j = 0; j < nbComments; j++) {
      const comment: Comment = {
        _id: `comment_${Date.now()}_${Math.random()}`,
        postId: postIds[i]!,
        content: commentTexts[Math.floor(Math.random() * commentTexts.length)]!,
        author: 'Toi',
        type: 'comment',
        creation_date: new Date(Date.now() - Math.random() * 7 * 24 * 60 * 60 * 1000).toISOString(),
      }

      try {
        await commentsDB.value.post(comment)
        totalComments++
      } catch (err: any) {
        console.error('Erreur création commentaire test:', err)
      }
    }
  }

  console.log('=> ' + totalComments + ' commentaires créés')
  console.log('=> Génération terminée')
  fetchData()
}

// ===== SUPPRIMER TOUS LES POSTS =====
const deleteAllPosts = async () => {
  if (!confirm('Êtes-vous sûr ? Tous les posts seront supprimés !')) {
    return
  }

  console.log('=> Suppression définitive de tous les posts et commentaires...')

  try {
    // Récupérer tous les posts
    const postsResult = await postsDB.value.allDocs({ include_docs: true })
    console.log('Posts à supprimer:', postsResult.rows.length)

    // Récupérer tous les commentaires
    const commentsResult = await commentsDB.value.allDocs({ include_docs: true })
    console.log('Commentaires à supprimer:', commentsResult.rows.length)

    // Supprimer tous les posts un par un
    for (const row of postsResult.rows) {
      try {
        await postsDB.value.remove(row.doc)
        console.log('Post supprimé:', row.id)
      } catch (err: any) {
        console.error('Erreur suppression post:', row.id, err)
      }
    }

    // Supprimer tous les commentaires un par un
    for (const row of commentsResult.rows) {
      try {
        await commentsDB.value.remove(row.doc)
        console.log('Commentaire supprimé:', row.id)
      } catch (err: any) {
        console.error('Erreur suppression commentaire:', row.id, err)
      }
    }

    console.log('✅ Tous les posts et commentaires supprimés (local + serveur)')
    postsData.value = []

    // Rafraîchir l'affichage
    await fetchData()
  } catch (err: any) {
    console.error('Erreur suppression posts:', err)
  }
}

// ===== GESTION DES CONFLITS =====
const detectConflicts = async () => {
  try {
    // Vérifier les conflits dans les posts
    const postsResult = await postsDB.value.allDocs({
      include_docs: true,
      conflicts: true,
    })

    // Vérifier les conflits dans les commentaires
    const commentsResult = await commentsDB.value.allDocs({
      include_docs: true,
      conflicts: true,
    })

    let conflictCount = 0
    const conflictedDocs: string[] = []

    // Filtrer les documents système
    const validPostRows = postsResult.rows.filter((row: any) => !row.id.startsWith('_'))
    validPostRows.forEach((row: any) => {
      if (row.doc._conflicts && row.doc._conflicts.length > 0) {
        conflictCount++
        conflictedDocs.push('Post: ' + (row.doc.title || row.doc._id))
        console.warn('⚠️ CONFLIT DÉTECTÉ sur post:', row.doc._id)
        console.warn('  Révision gagnante:', row.doc._rev)
        console.warn('  Révisions perdantes:', row.doc._conflicts)
      }
    })

    const validCommentRows = commentsResult.rows.filter((row: any) => !row.id.startsWith('_'))
    validCommentRows.forEach((row: any) => {
      if (row.doc._conflicts && row.doc._conflicts.length > 0) {
        conflictCount++
        conflictedDocs.push('Commentaire: ' + row.doc._id)
        console.warn('⚠️ CONFLIT DÉTECTÉ sur commentaire:', row.doc._id)
        console.warn('  Révision gagnante:', row.doc._rev)
        console.warn('  Révisions perdantes:', row.doc._conflicts)
      }
    })

    if (conflictCount > 0) {
      alert(
        `⚠️ ${conflictCount} conflit(s) de fusion détecté(s):\n\n${conflictedDocs.join('\n')}\n\nLes données ont été fusionnées automatiquement. Vérifiez le contenu de ces éléments.`,
      )
    }
  } catch (err: any) {
    console.error('Erreur détection conflits:', err)
  }
}

const resolveConflicts = async () => {
  try {
    // Résoudre les conflits des posts
    const postsResult = await postsDB.value.allDocs({
      include_docs: true,
      conflicts: true,
    })

    // Filtrer les documents système
    const validPostRows = postsResult.rows.filter((row: any) => !row.id.startsWith('_'))
    for (const row of validPostRows) {
      if (row.doc._conflicts && row.doc._conflicts.length > 0) {
        console.log('=> Résolution du conflit sur post:', row.doc._id)

        const losingRevs = row.doc._conflicts

        for (const conflictRev of losingRevs) {
          try {
            await postsDB.value.remove({
              _id: row.doc._id,
              _rev: conflictRev,
            })
            console.log('✓ Conflit post résolu pour:', row.doc._id)
          } catch (err: any) {
            console.error('Erreur suppression conflit post:', err)
          }
        }
      }
    }

    // Résoudre les conflits des commentaires
    const commentsResult = await commentsDB.value.allDocs({
      include_docs: true,
      conflicts: true,
    })

    // Filtrer les documents système
    const validCommentRows = commentsResult.rows.filter((row: any) => !row.id.startsWith('_'))
    for (const row of validCommentRows) {
      if (row.doc._conflicts && row.doc._conflicts.length > 0) {
        console.log('=> Résolution du conflit sur commentaire:', row.doc._id)

        const losingRevs = row.doc._conflicts

        for (const conflictRev of losingRevs) {
          try {
            await commentsDB.value.remove({
              _id: row.doc._id,
              _rev: conflictRev,
            })
            console.log('✓ Conflit commentaire résolu pour:', row.doc._id)
          } catch (err: any) {
            console.error('Erreur suppression conflit commentaire:', err)
          }
        }
      }
    }
  } catch (err: any) {
    console.error('Erreur résolution conflits:', err)
  }
}

const searchReset = () => {
  const searchInput = document.querySelector('.search') as HTMLInputElement
  if (searchInput) searchInput.value = ''
  fetchData()
}

const toggle = () => {
  if (!offlineMode.value) {
    postsSync.value.cancel()
    commentsSync.value.cancel()
    postsSync.value = null
    commentsSync.value = null
    offlineMode.value = true
    syncStatus.value = '📵 Mode hors ligne (simulation)'
    isOnline.value = false
    offlineChanges.value = 0
    console.log('=> Sync arrêtée - Mode hors ligne activé')
  } else {
    // Réactiver la sync
    offlineMode.value = false
    offlineChanges.value = 0
    syncStatus.value = '🔄 Reconnexion...'
    syncData()

    setTimeout(async () => {
      await detectConflicts()
      fetchData()
    }, 2000)

    console.log('=> Mode hors ligne désactivé - Sync réactivée')
  }
}

// ===== MODE HORS LIGNE (SIMULATION VIA TOGGLE) =====
const trackLocalChange = () => {
  if (offlineMode.value) {
    offlineChanges.value++
    console.log('=> Changement local enregistré (non synchronisé):', offlineChanges.value)
  }
}
</script>

<template>
  <h1>Fetch Data</h1>

  <!-- STATUT DE RÉPLICATION -->
  <div class="replication-status">
    <p>{{ replicationStatus }}</p>
  </div>

  <div>
    <label v-if="!offlineMode && isOnline" style="color: green; font-weight: bold">
      🟢 {{ syncStatus }}
    </label>
    <label v-else-if="offlineMode" style="color: #ff6b6b; font-weight: bold">
      {{ syncStatus }}
    </label>
    <label v-else style="color: red; font-weight: bold"> 🔴 {{ syncStatus }} </label>
    <input
      @click="toggle"
      type="checkbox"
      name="toggleSync"
      :checked="postsSync != null && commentsSync != null"
    />
    <label for="toggleSync">Toggle Sync</label>

    <span v-if="lastSyncTime && !offlineMode" style="font-size: 0.9em; color: #999">
      (Dernière sync: {{ lastSyncTime }})
    </span>

    <span
      v-if="offlineMode && offlineChanges > 0"
      style="font-size: 0.9em; color: #ff6b6b; font-weight: bold"
    >
      | ⚠️ {{ offlineChanges }} changement(s) en attente
    </span>
  </div>
  <div>
    <input type="text" placeholder="Search" @keyup.enter="search" class="search" />
    <button @click="searchReset">✕ Réinitialiser</button>
    <button @click="sortByLikes" class="btn-sort">📊 Trier par likes</button>
  </div>

  <!-- SECTION CRÉATION POST -->
  <div class="create-section">
    <h2>📝 Créer un post</h2>
    <input type="text" class="input-title" placeholder="Titre du post" />
    <textarea class="input-content" placeholder="Contenu du post"></textarea>
    <button @click="createDoc" class="btn-primary">➕ Ajouter un post</button>
    <button @click="generateTestData" class="btn-secondary">
      🧪 Générer données test (15 posts)
    </button>
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
    <button @click="toggleLike(post)" class="btn-like">👍 {{ post.likes }} likes</button>

    <!-- SECTION COMMENTAIRES -->
    <div class="comments-section">
      <h3>💬 Commentaires ({{ (post.comments ?? []).length }})</h3>
      <div v-for="comment in post.comments" :key="comment._id" class="comment">
        <strong>{{ comment.author }}</strong>
        <span class="comment-date">({{ new Date(comment.creation_date).toLocaleString() }})</span>
        <p>{{ comment.content }}</p>
        <button @click="deleteComment(post, comment)" class="btn-comment-delete">✕</button>
        <button @click="updateComment(post, comment)" class="btn-comment-edit">✏️</button>
      </div>
      <div class="comment-input-wrapper">
        <input
          type="text"
          :class="`comment-input-${post._id}`"
          placeholder="Ajouter un commentaire"
          class="comment-input"
        />
        <button @click="addComment(post)" class="btn-comment">💬 Commenter</button>
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
input[type='text'],
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

input[type='checkbox'] {
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

.btn-primary {
  background-color: #28a745;
  color: white;
}

.btn-primary:hover {
  background-color: #218838;
}

.btn-secondary {
  background-color: #6c757d;
  color: white;
}

.btn-secondary:hover {
  background-color: #5a6268;
}

.btn-sort {
  background-color: #17a2b8;
  color: white;
}

.btn-sort:hover {
  background-color: #138496;
}

.btn-small {
  background-color: #007bff;
  padding: 6px 10px;
  font-size: 0.9em;
  margin-right: 5px;
}

.btn-small:hover {
  background-color: #0056b3;
}

.btn-small-danger {
  background-color: #dc3545;
  padding: 6px 10px;
  font-size: 0.9em;
  margin-right: 5px;
}

.btn-small-danger:hover {
  background-color: #c82333;
}

.btn-like {
  background-color: #ffc107;
  color: #333;
  padding: 8px 12px;
}

.btn-like:hover {
  background-color: #e0a800;
}

.btn-comment {
  background-color: #007bff;
  color: white;
  padding: 8px 12px;
}

.btn-comment:hover {
  background-color: #0056b3;
}

.replication-status {
  background-color: #e8f4f8;
  border: 1px solid #3498db;
  border-radius: 4px;
  padding: 12px;
  margin-bottom: 15px;
  text-align: center;
  color: #2c3e50;
  font-weight: bold;
}

.replication-status p {
  margin: 0;
  color: #2c3e50;
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

.btn-comment-delete {
  background-color: #e74c3c;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 6px 10px;
  cursor: pointer;
}

.btn-comment-delete:hover {
  background-color: #c0392b;
}

.btn-comment-edit {
  background-color: #ffc107;
  color: #333;
  border: none;
  border-radius: 4px;
  padding: 6px 10px;
  cursor: pointer;
}

.btn-comment-edit:hover {
  background-color: #e0a800;
}
</style>
