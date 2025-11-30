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
let counter = 0;

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

    localdb.replicate.from(url)
    .on('complete', syncData)
    fetchData()
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
    .allDocs({ include_docs: true, attachments: true })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.rows)
      postsData.value = result.rows.map((row: any) => row.doc)
    })
    .catch((error: any) => {
      console.error('Erreur lors de la récupération des données :', error)
    })
}

const createDoc = (): any => {
  counter++
  storage.value
    .post({
      title: 'Document' + counter,
      content: 'Contenu du document ',
    })
    .then(function (response: any) {
      fetchData()
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const deleteDoc = (post: any): any => {
  storage.value
    .remove(post)
    .then(function (response: any) {
      fetchData()
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const updateDoc = (post: any): any => {
  post.content = post.content + ' (modifié)'
  storage.value
    .put(post)
    .then((response: any) => {
      fetchData()
      console.log(response)
    })
    .catch((err: any) => {
      console.log(err)
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
      console.log('✓ Post ' + i + ' créé')
    } catch (err: any) {
      console.error('Erreur création post test:', err)
    }
  }

  console.log('=> Génération terminée');
  fetchData()
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
  <div>
    <button @click="createDoc">Ajouter un document</button>
    <button @click="generateTestData">🧪 Générer données test (15 posts)</button>
  </div>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.title }}</h2>
    <p>{{ post.content }}</p>
    <p>👍 {{ post.likes }} likes</p>
    <button @click="deleteDoc(post)">Supprimer le document</button>
    <button @click="updateDoc(post)">Modifier le document</button>
  </article>
</template>
