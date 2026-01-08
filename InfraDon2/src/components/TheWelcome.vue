<script setup lang="ts">
// Application de gestion de posts et commentaires avec PouchDB/CouchDB
// 2 bases séparées : posts et comments (pour meilleures performances)
// Sync bidirectionnelle en temps réel + mode offline

import PouchDB from 'pouchdb'
import { onMounted, ref } from 'vue'
import findPlugin from 'pouchdb-find'
PouchDB.plugin(findPlugin)

interface Comment {
  _id: string
  _rev?: string
  postId: string
  content: string
  author: string
  type: 'comment'
  creation_date: string
}

interface Post {
  _id: string
  _rev?: string
  type: 'post'
  title: string
  content: string
  likes: number
  comments?: Comment[]
  commentCount?: number
  creation_date: string
  updated_date: string
  _attachments?: Record<string, {
    content_type: string
    data?: string | Blob
    stub?: boolean
    length?: number
  }>
}

// Connexions aux bases de données
const postsDB = ref()
const commentsDB = ref()
const postsUrl = 'http://inoe.wenger:IWtramp54HEIG/@localhost:5984/infradon_inoe_posts/'
const commentsUrl = 'http://inoe.wenger:IWtramp54HEIG/@localhost:5984/infradon_inoe_comments/'
const opts = { live: true, retry: true }
const postsData = ref<Post[]>([])
const postsSync = ref()
const commentsSync = ref()
const replicationStatus = ref('En attente...')
const isOnline = ref(true)
const syncStatus = ref('Synchronisé')
const lastSyncTime = ref('')
const offlineMode = ref(false)

const currentPage = ref(0)
const postsPerPage = ref(10)
const totalPosts = ref(0)
const hasMorePosts = ref(true)
const isLoading = ref(false)

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
    await createDesignDocuments() // Créer les vues MapReduce
    replicateFromServer()
  } else {
    console.warn('Something went wrong')
  }
}

const replicateFromServer = () => {
  console.log('=> Début de la réplication depuis le serveur')
  replicationStatus.value = '⏳ Réplication en cours...'

  // Tout répliquer pour mode offline complet
  // Pour projet réel avec beaucoup de données : envisager filtrage

  let postsReplicationDone = false
  let commentsReplicationDone = false

  const checkBothComplete = () => {
    if (postsReplicationDone && commentsReplicationDone) {
      console.log('Réplication des 2 bases complète')
      replicationStatus.value = '✅ Synchronisation complète'
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
    .on('error', (err: Error) => {
      console.error('Erreur réplication posts :', err)
      replicationStatus.value = 'Erreur posts : ' + err.message
    })

  // Réplication des commentaires
  commentsDB.value.replicate
    .from(commentsUrl)
    .on('change', (info: { docs_read: number }) => {
      console.log('=> Commentaires : ' + info.docs_read + ' documents lus')
    })
    .on('complete', (info: { docs_read: number }) => {
      console.log('Réplication commentaires complète :', info.docs_read + ' documents')
      commentsReplicationDone = true
      checkBothComplete()
    })
    .on('error', (err: Error) => {
      console.error('Erreur réplication commentaires :', err)
      replicationStatus.value = 'Erreur commentaires : ' + err.message
    })
}

const syncData = () => {
  if (offlineMode.value) {
    console.log('=> Pas de sync (mode hors ligne activé)')
    return
  }

  // sync() = bidirectionnel (local <-> serveur) en temps réel
  // live:true = surveille les changements en continu
  // retry:true = reconnexion automatique si perte de connexion

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

const searchQuery = ref('')

const search = async () => {
  if (!searchQuery.value.trim()) {
    await fetchData(0, postsPerPage.value)
    return
  }

  isLoading.value = true

  try {
    const result = await postsDB.value.find({
      selector: {
        $or: [
          { title: { $regex: searchQuery.value.trim() } },
          { content: { $regex: searchQuery.value.trim() } }
        ],
      },
    })

    const postsWithComments = await attachLastComment(result.docs)
    postsData.value = postsWithComments
  } catch (error) {
    console.error('Erreur lors de la recherche :', error)
  }

  isLoading.value = false
}

const createIndexes = async () => {
  // Index nécessaires pour la recherche avec find()
  try {
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

// Vues MapReduce pour optimiser les requêtes
// Plus rapide que allDocs car indexé et mis en cache
const createDesignDocuments = async () => {
  console.log('=> Création des design documents (vues MapReduce)')

  // Vue posts : tri par likes et par date
  const postsByLikesView: any = {
    _id: '_design/posts_views',
    views: {
      by_likes: {
        map: `function(doc) {
          if (doc.type === 'post') {
            emit(doc.likes, doc);
          }
        }`.toString(),
      },
      by_date: {
        map: `function(doc) {
          if (doc.type === 'post') {
            emit(doc.creation_date, doc);
          }
        }`.toString(),
      },
      count_posts: {
        map: `function(doc) {
          if (doc.type === 'post') {
            emit(doc._id, 1);
          }
        }`.toString(),
        reduce: '_count',
      },
    },
  }

  // Vue commentaires : récupérer par post + date
  const commentsByPostView: any = {
    _id: '_design/comments_views',
    views: {
      by_post_and_date: {
        map: `function(doc) {
          if (doc.type === 'comment') {
            emit([doc.postId, doc.creation_date], doc);
          }
        }`.toString(),
      },
      count_by_post: {
        map: `function(doc) {
          if (doc.type === 'comment') {
            emit(doc.postId, 1);
          }
        }`.toString(),
        reduce: '_count',
      },
    },
  }

  try {
    // Créer ou mettre à jour la vue des posts
    try {
      const existingPostsView = await postsDB.value.get('_design/posts_views')
      postsByLikesView._rev = existingPostsView._rev
      await postsDB.value.put(postsByLikesView)
      console.log('Vue posts_views mise à jour')
    } catch (err: any) {
      if (err.status === 404) {
        await postsDB.value.put(postsByLikesView)
        console.log('Vue posts_views créée')
      }
    }

    // Créer ou mettre à jour la vue des commentaires
    try {
      const existingCommentsView = await commentsDB.value.get('_design/comments_views')
      commentsByPostView._rev = existingCommentsView._rev
      await commentsDB.value.put(commentsByPostView)
      console.log('Vue comments_views mise à jour')
    } catch (err: any) {
      if (err.status === 404) {
        await commentsDB.value.put(commentsByPostView)
        console.log('Vue comments_views créée')
      }
    }
  } catch (err: any) {
    console.error('Erreur création design documents:', err)
  }
}

// Top 10 posts likés avec pagination
const fetchTopLikedPosts = async (page: number = 0): Promise<any> => {
  console.log(`=> Récupération des 10 posts les plus likés (page ${page})`)
  isLoading.value = true
  currentPage.value = page

  try {
    const skip = page * postsPerPage.value
    const result = await postsDB.value.query('posts_views/by_likes', {
      descending: true,
      skip: skip,
      limit: postsPerPage.value,
      include_docs: false,
    })

    console.log(`=> ${result.rows.length} posts récupérés (skip: ${skip})`)
    const posts = result.rows.map((row: any) => row.value) as Post[]
    hasMorePosts.value = result.rows.length === postsPerPage.value

    const postsWithComments = await attachLastComment(posts)
    postsData.value = postsWithComments as Post[]

    // Charger les URLs des attachments
    for (const post of postsWithComments) {
      await loadPostAttachments(post)
    }
  } catch (error: any) {
    console.error('Erreur fetchTopLikedPosts:', error)
  }

  isLoading.value = false
}

const nextPage = () => {
  if (hasMorePosts.value) {
    fetchTopLikedPosts(currentPage.value + 1)
  }
}

const previousPage = () => {
  if (currentPage.value > 0) {
    fetchTopLikedPosts(currentPage.value - 1)
  }
}

const resetPagination = () => {
  currentPage.value = 0
  fetchData(0, postsPerPage.value)
}

// Charger les posts avec pagination via vue MapReduce
// Évite allDocs() qui charge tout en mémoire
const fetchData = async (skip: number = 0, limit: number = 10): Promise<any> => {
  isLoading.value = true

  try {
    // Vue 'by_date' pour posts triés par date (plus récents en premier)
    const result = await postsDB.value.query('posts_views/by_date', {
      descending: true,
      skip: skip,
      limit: limit,
      include_docs: false,
    })

    const posts = result.rows.map((row: any) => row.value) as Post[]

    // Vérifier s'il y a plus de posts
    hasMorePosts.value = result.rows.length === limit

    // Ajouter les commentaires (optimisé pour ne charger que le dernier)
    const postsWithComments = await attachLastComment(posts)
    postsData.value = postsWithComments as Post[]

    // Charger les URLs des attachments
    for (const post of postsWithComments) {
      await loadPostAttachments(post)
    }

    totalPosts.value = result.total_rows || posts.length
  } catch (error: any) {
    console.error('Erreur lors de la récupération des données :', error)
    // Fallback : Si les vues n'existent pas encore, utiliser find()
    console.warn('Fallback : Utilisation de find() au lieu de query()')
    const fallbackResult = await postsDB.value.find({
      selector: { type: 'post' },
      sort: [{ creation_date: 'desc' }],
      skip: skip,
      limit: limit,
    })
    const postsWithComments = await attachLastComment(fallbackResult.docs)
    postsData.value = postsWithComments as Post[]

    // Charger les URLs des attachments
    for (const post of postsWithComments) {
      await loadPostAttachments(post)
    }
  }

  isLoading.value = false
}

// Charger uniquement le dernier commentaire par défaut
// Gain de performance : évite de charger 100 comments si pas nécessaire
const attachLastComment = async (posts: Post[]): Promise<Post[]> => {
  const results = await Promise.all(
    posts.map(async (post) => {
      try {
        const commentsResult = await commentsDB.value.query('comments_views/by_post_and_date', {
          startkey: [post._id, {}],
          endkey: [post._id],
          descending: true,
          limit: 1,
          include_docs: false,
        })

        const countResult = await commentsDB.value.query('comments_views/count_by_post', {
          key: post._id,
          reduce: true,
        })

        const lastComment = commentsResult.rows.length > 0 ? commentsResult.rows[0].value : null
        const commentCount = countResult.rows.length > 0 ? countResult.rows[0].value : 0

        return {
          ...post,
          comments: lastComment ? [lastComment] : [],
          commentCount: commentCount,
        }
      } catch (err: any) {
        console.error('Erreur attachLastComment pour post', post._id, err)
        return { ...post, comments: [], commentCount: 0 }
      }
    }),
  )
  return results
}

const newPostTitle = ref('')
const newPostContent = ref('')

const createDoc = async () => {
  if (!newPostTitle.value.trim() || !newPostContent.value.trim()) {
    console.warn('Titre et contenu sont obligatoires')
    return
  }

  const newPost: Post = {
    _id: `post_${Date.now()}`,
    type: 'post',
    title: newPostTitle.value.trim(),
    content: newPostContent.value.trim(),
    likes: 0,
    creation_date: new Date().toISOString(),
    updated_date: new Date().toISOString(),
  }

  try {
    await postsDB.value.post(newPost)
    newPostTitle.value = ''
    newPostContent.value = ''
    await fetchData(0, postsPerPage.value)
  } catch (err) {
    console.error('Erreur création post :', err)
  }
}

const deleteDoc = async (post: Post) => {
  try {
    await postsDB.value.remove(post)
    await fetchData(0, postsPerPage.value)
  } catch (err) {
    console.error('Erreur suppression post :', err)
  }
}

const updateDoc = async (post: Post) => {
  const newTitle = prompt('Nouveau titre:', post.title)
  if (!newTitle?.trim()) return

  const newContent = prompt('Nouveau contenu:', post.content)
  if (!newContent?.trim()) return

  post.title = newTitle.trim()
  post.content = newContent.trim()
  post.updated_date = new Date().toISOString()

  try {
    await postsDB.value.put(post)
    await fetchData(0, postsPerPage.value)
  } catch (err) {
    console.error('Erreur modification post :', err)
  }
}

const toggleLike = async (post: Post) => {
  post.likes++
  post.updated_date = new Date().toISOString()

  try {
    await postsDB.value.put(post)
    await fetchData(0, postsPerPage.value)
  } catch (err) {
    console.error('Erreur modification like :', err)
  }
}

const commentInputs = ref<Record<string, string>>({})

const addComment = async (post: Post) => {
  const content = commentInputs.value[post._id]?.trim()
  if (!content) return

  const newComment: Comment = {
    _id: `comment_${Date.now()}`,
    postId: post._id,
    content,
    author: 'Toi',
    type: 'comment',
    creation_date: new Date().toISOString(),
  }

  try {
    await commentsDB.value.post(newComment)
    commentInputs.value[post._id] = ''
    await fetchData(currentPage.value * postsPerPage.value, postsPerPage.value)
  } catch (err) {
    console.error('Erreur ajout commentaire :', err)
  }
}

const deleteComment = async (post: Post, comment: Comment) => {
  try {
    await commentsDB.value.remove(comment)
    await fetchData(currentPage.value * postsPerPage.value, postsPerPage.value)
  } catch (err) {
    console.error('Erreur suppression commentaire :', err)
  }
}

const updateComment = async (post: Post, comment: Comment) => {
  const newContent = prompt('Nouveau contenu du commentaire:', comment.content)
  if (!newContent?.trim()) return

  comment.content = newContent.trim()

  try {
    await commentsDB.value.put(comment)
    await fetchData(currentPage.value * postsPerPage.value, postsPerPage.value)
  } catch (err) {
    console.error('Erreur modification commentaire :', err)
  }
}

// Générer 15 posts et commentaires aléatoires pour tester
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

  // Petit délai pour laisser la sync se faire
  await new Promise(resolve => setTimeout(resolve, 500))
  await fetchData(0, postsPerPage.value)
}

// Supprimer tous les posts et commentaires
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
    await fetchData(0, postsPerPage.value)
  } catch (err: any) {
    console.error('Erreur suppression posts:', err)
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
  searchQuery.value = ''
  fetchData(0, postsPerPage.value)
}

const toggle = () => {
  if (!offlineMode.value) {
    postsSync.value.cancel()
    commentsSync.value.cancel()
    postsSync.value = null
    commentsSync.value = null
    offlineMode.value = true
    syncStatus.value = '📵 Mode hors ligne'
    isOnline.value = false
  } else {
    offlineMode.value = false
    syncStatus.value = '🔄 Reconnexion...'
    syncData()

    setTimeout(async () => {
      await resolveConflicts()
      await fetchData(0, postsPerPage.value)
    }, 2000)
  }
}



// Charger tous les commentaires d'un post à la demande
const loadAllComments = async (post: Post): Promise<void> => {
  try {
    const result = await commentsDB.value.query('comments_views/by_post_and_date', {
      startkey: [post._id, {}],
      endkey: [post._id],
      descending: true,
      include_docs: false,
    })

    const allComments = result.rows.map((row: any) => row.value) as Comment[]

    const postIndex = postsData.value.findIndex((p) => p._id === post._id)
    if (postIndex !== -1 && postsData.value[postIndex]) {
      postsData.value[postIndex]!.comments = allComments
    }
  } catch (err) {
    console.error('Erreur chargement commentaires:', err)
  }
}

// Ajouter un fichier (image/vidéo) à un post
const addAttachment = async (post: Post): Promise<void> => {
  const input = document.createElement('input')
  input.type = 'file'
  input.accept = 'image/*,video/*'

  input.onchange = async (e: any) => {
    const file = e.target?.files?.[0]
    if (!file) return

    console.log('Fichier sélectionné:', file.name, file.type, file.size)

    if (file.size > 5 * 1024 * 1024) {
      alert('⚠️ Fichier trop volumineux (max 5MB). Pour des fichiers plus gros, utilisez un service cloud.')
      return
    }

    try {
      const latestPost = await postsDB.value.get(post._id)

      const response = await postsDB.value.putAttachment(
        latestPost._id,
        file.name,
        latestPost._rev!,
        file,
        file.type,
      )

      console.log('Attachment ajouté:', response)
      await fetchData(currentPage.value * postsPerPage.value, postsPerPage.value)
    } catch (err: any) {
      console.error('Erreur ajout attachment:', err)
      alert('Erreur lors de l\'ajout du fichier: ' + err.message)
    }
  }

  input.click()
}

const removeAttachment = async (post: Post, attachmentName: string): Promise<void> => {
  if (!confirm(`Supprimer le fichier "${attachmentName}" ?`)) {
    return
  }

  try {
    const latestPost = await postsDB.value.get(post._id)

    const response = await postsDB.value.removeAttachment(
      latestPost._id,
      attachmentName,
      latestPost._rev!,
    )

    console.log('Attachment supprimé:', response)
    await fetchData(currentPage.value * postsPerPage.value, postsPerPage.value)
  } catch (err: any) {
    console.error('Erreur suppression attachment:', err)
    alert('Erreur lors de la suppression: ' + err.message)
  }
}

// Cache pour les URLs blob des attachments
const attachmentUrls = ref<Record<string, string>>({})

// Charger l'URL d'un attachment et la mettre en cache
const loadAttachmentUrl = async (post: Post, attachmentName: string) => {
  const cacheKey = `${post._id}_${attachmentName}`

  if (attachmentUrls.value[cacheKey]) {
    return
  }

  try {
    const blob = await postsDB.value.getAttachment(post._id, attachmentName)
    const url = URL.createObjectURL(blob)
    attachmentUrls.value[cacheKey] = url
  } catch (err) {
    console.error('Erreur chargement attachment:', err)
  }
}

const getAttachmentUrl = (post: Post, attachmentName: string): string => {
  const cacheKey = `${post._id}_${attachmentName}`
  return attachmentUrls.value[cacheKey] || ''
}

// Charger les URLs de tous les attachments d'un post
const loadPostAttachments = async (post: Post) => {
  if (post._attachments) {
    for (const filename of Object.keys(post._attachments)) {
      await loadAttachmentUrl(post, filename)
    }
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
  </div>
  <div>
    <input type="text" v-model="searchQuery" placeholder="Search" @keyup.enter="search" class="search" />
    <button @click="searchReset">✕ Réinitialiser</button>
    <button @click="fetchTopLikedPosts(0)" class="btn-sort">🔥 Top 10 likés</button>
    <button @click="resetPagination" class="btn-sort">🔄 Tous les posts</button>
  </div>

  <!-- CONTRÔLES DE PAGINATION -->
  <div class="pagination-controls" v-if="!isLoading">
    <button @click="previousPage" :disabled="currentPage === 0" class="btn-pagination">
      ← Précédent
    </button>
    <span class="page-info">Page {{ currentPage + 1 }}</span>
    <button @click="nextPage" :disabled="!hasMorePosts" class="btn-pagination">
      Suivant →
    </button>
  </div>

  <!-- INDICATEUR DE CHARGEMENT -->
  <div v-if="isLoading" class="loading-indicator">
    <p>⏳ Chargement en cours...</p>
  </div>

  <!-- SECTION CRÉATION POST -->
  <div class="create-section">
    <h2>📝 Créer un post</h2>
    <input type="text" v-model="newPostTitle" placeholder="Titre du post" />
    <textarea v-model="newPostContent" placeholder="Contenu du post"></textarea>
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
      <button @click="addAttachment(post)" class="btn-small-media">📎 Ajouter média</button>
    </div>

    <!-- SECTION MÉDIAS/ATTACHMENTS -->
    <div v-if="post._attachments && Object.keys(post._attachments).length > 0" class="media-section">
      <h4>🖼️ Médias attachés ({{ Object.keys(post._attachments).length }})</h4>
      <div class="media-grid">
        <div
          v-for="(attachment, filename) in post._attachments"
          :key="filename"
          class="media-item"
        >
          <img
            v-if="attachment.content_type?.startsWith('image/')"
            :src="getAttachmentUrl(post, filename as string)"
            :alt="filename as string"
            class="media-image"
          />
          <div v-else-if="attachment.content_type?.startsWith('video/')" class="media-video">
            <video controls class="media-video-player">
              <source :src="getAttachmentUrl(post, filename as string)" :type="attachment.content_type" />
              Votre navigateur ne supporte pas la vidéo.
            </video>
          </div>
          <div v-else class="media-file">
            📄 {{ filename }}
          </div>
          <div class="media-info">
            <p class="media-filename">{{ filename }}</p>
            <button @click="removeAttachment(post, filename as string)" class="btn-remove-media">
              🗑️
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- BOUTON LIKES -->
    <button @click="toggleLike(post)" class="btn-like">👍 {{ post.likes }} likes</button>

    <!-- SECTION COMMENTAIRES -->
    <div class="comments-section">
      <h3>
        💬 Commentaires
        <span v-if="post.commentCount !== undefined" class="comment-count">
          ({{ post.commentCount }} au total)
        </span>
        <span v-else class="comment-count">({{ (post.comments ?? []).length }})</span>
      </h3>

      <!-- Afficher seulement le dernier commentaire par défaut -->
      <div v-if="(post.comments ?? []).length === 1 && post.commentCount && post.commentCount > 1">
        <p class="last-comment-indicator">🔽 Dernier commentaire :</p>
      </div>

      <div v-for="comment in post.comments" :key="comment._id" class="comment">
        <strong>{{ comment.author }}</strong>
        <span class="comment-date">({{ new Date(comment.creation_date).toLocaleString() }})</span>
        <p>{{ comment.content }}</p>
        <button @click="deleteComment(post, comment)" class="btn-comment-delete">✕</button>
        <button @click="updateComment(post, comment)" class="btn-comment-edit">✏️</button>
      </div>

      <!-- Bouton pour charger tous les commentaires -->
      <button
        v-if="post.commentCount && post.commentCount > 1 && (post.comments ?? []).length < post.commentCount"
        @click="loadAllComments(post)"
        class="btn-load-comments"
      >
        👁️ Voir tous les {{ post.commentCount }} commentaires
      </button>

      <div class="comment-input-wrapper">
        <input
          type="text"
          v-model="commentInputs[post._id]"
          placeholder="Ajouter un commentaire"
          class="comment-input"
          @keyup.enter="addComment(post)"
        />
        <button @click="addComment(post)" class="btn-comment">💬 Commenter</button>
      </div>
    </div>
  </article>
</template>

<style scoped>
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

.btn-small-media {
  background-color: #28a745;
  padding: 6px 10px;
  font-size: 0.9em;
  margin-right: 5px;
  color: white;
}

.btn-small-media:hover {
  background-color: #218838;
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

.btn-load-comments {
  background-color: #17a2b8;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 8px 12px;
  cursor: pointer;
  margin-top: 10px;
  width: 100%;
}

.btn-load-comments:hover {
  background-color: #138496;
}

/* ===== PAGINATION ===== */
.pagination-controls {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 15px;
  margin: 20px 0;
  padding: 15px;
  background-color: #f8f9fa;
  border-radius: 4px;
}

.btn-pagination {
  background-color: #007bff;
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}

.btn-pagination:hover:not(:disabled) {
  background-color: #0056b3;
}

.btn-pagination:disabled {
  background-color: #ccc;
  cursor: not-allowed;
  opacity: 0.6;
}

.page-info {
  font-weight: bold;
  color: #333;
  padding: 0 10px;
}

/* ===== INDICATEUR DE CHARGEMENT ===== */
.loading-indicator {
  text-align: center;
  padding: 30px;
  background-color: #e8f4f8;
  border-radius: 4px;
  margin: 20px 0;
}

.loading-indicator p {
  color: #2c3e50;
  font-size: 1.2em;
  font-weight: bold;
  margin: 0;
}

/* ===== COMPTEURS ===== */
.comment-count {
  color: #666;
  font-weight: normal;
  font-size: 0.9em;
}

.last-comment-indicator {
  color: #999;
  font-size: 0.85em;
  font-style: italic;
  margin: 5px 0;
}

/* ===== MÉDIAS / ATTACHMENTS ===== */
.media-section {
  background-color: #f5f5f5;
  padding: 15px;
  margin: 15px 0;
  border-radius: 4px;
  border: 1px solid #ddd;
}

.media-section h4 {
  margin-top: 0;
  color: #333;
}

.media-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 15px;
  margin-top: 10px;
}

.media-item {
  background-color: white;
  border: 1px solid #ddd;
  border-radius: 4px;
  overflow: hidden;
}

.media-image {
  width: 100%;
  height: 200px;
  object-fit: cover;
  display: block;
}

.media-video {
  width: 100%;
  background-color: #000;
}

.media-video-player {
  width: 100%;
  height: 200px;
}

.media-file {
  width: 100%;
  height: 200px;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #e9ecef;
  font-size: 2em;
}

.media-info {
  padding: 10px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  background-color: #f8f9fa;
}

.media-filename {
  margin: 0;
  font-size: 0.85em;
  color: #555;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  flex: 1;
}

.btn-remove-media {
  background-color: #dc3545;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 5px 10px;
  cursor: pointer;
  font-size: 0.9em;
}

.btn-remove-media:hover {
  background-color: #c82333;
}
</style>
