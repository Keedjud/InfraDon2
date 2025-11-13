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
let counter = 0;

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase()
});

const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('local')
  if (db) {
    console.log("Connected to collection : " + db?.name)
    storage.value = db
    startSync(db)
    fetchData()
  } else {
    console.warn('Something went wrong')
  }
  return db
}

const startSync = (db: any) => {
  if (!db) return
  db.sync(url, opts)
    .on('change', (change: any) => {
      console.log('sync change', change)
      fetchData()
    })
}
const toggle = () => {

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
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.title }}</h2>
    <p>{{ post.content }}</p>
    <button @click="deleteDoc(post)">Supprimer le document</button>
    <button @click="updateDoc(post)">Modifier le document</button>
  </article>
  <button @click="createDoc">Ajouter un document</button>
  <div>
    <input @click="toggle" type="checkbox" name="toggleSync" checked />
    <label for="toggleSync">Toggle Sync</label>
  </div>
</template>
