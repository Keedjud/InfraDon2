<script setup lang="ts">
import PouchDB from 'pouchdb'
import { onMounted, ref } from 'vue';

const storage = ref();

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase()
});

const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://inoe.wenger:IWtramp54HEIG/@localhost:5984/')
  if (db) {
    console.log("Connected to collection : " + db?.name)
    storage.value = db
  } else {
    console.warn('Something went wrong')
  }
}

// DATA - MODEL
const counter = ref(20)

// METHODS - CONTROLLER
const increment = () => {
  counter.value++
}

</script>

<template>
  <h1>Hello World</h1>
  <button @click="increment">Increment</button>
  <p style="color: white; font-size: 50px;">{{ counter }}</p>
</template>
