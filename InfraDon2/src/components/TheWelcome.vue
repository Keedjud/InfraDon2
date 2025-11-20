<script setup lang="ts">
import PouchDB from 'pouchdb'
import { onMounted, ref } from 'vue';

declare interface Post {
  _id: string
  _rev: string
  title: string
  content: string
  attributes: {
    creation_date: any
  }
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
    storage.value.createIndex({
      index: {
        fields: ['content']
      }
    })
    .then(console.log("index created"))
    localdb.replicate.from(url)
    .on('complete', syncData)
    fetchData()
  } else {
    console.warn('Something went wrong')
  }
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
  document.querySelector(".search").value = ""
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
  </div>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.title }}</h2>
    <p>{{ post.content }}</p>
    <button @click="deleteDoc(post)">Supprimer le document</button>
    <button @click="updateDoc(post)">Modifier le document</button>
  </article>
</template>
