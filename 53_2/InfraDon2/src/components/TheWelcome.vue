<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

import FindPlugin from 'pouchdb-find'
PouchDB.plugin(FindPlugin);

declare interface Post {
  number: number
  title: string
  content: string
  name: string
  attributes: {
    creation_date: any
  }
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
  console.log(postsData.value)
})

const storage = ref()
const postsData = ref<Post[]>([])
let counter = 0
const searchQuery = ref<string>('')

const isOffline = ref(false)
let liveSync: any = null

const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const localDB = new PouchDB('infradon_local')
  if (localDB) {
    console.log('Connected to collection : ' + localDB?.name)
    storage.value = localDB
  } else {
    console.warn('Something went wrong')
  }

  storage.value
  .then(function (response: any) {
      startSync()
      console.log(response)
    })
}

const indexNumber = () => {
  const num = parseInt(searchQuery.value)
  if (isNaN(num)) {
    fetchData()
    return
  }

  storage.value
    .allDocs({ include_docs: true })
    .then((result: any) => {
      postsData.value = result.rows.map((r: any) => r.doc).filter((doc: any) => doc.number === num)
    })
    .catch((err: any) => {
      console.error('Erreur recherche :', err)
    })
}

const fetchData = (): any => {
  storage.value
    .allDocs({
      include_docs: true,
    })
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
      number: counter,
      title: 'Document ' + counter,
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

  post.title = post.title + ' (modifié)'
  storage.value
    .put(post)
    .then(function (response: any) {
      fetchData()
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const toggleOffline = () => {
  isOffline.value = !isOffline.value

  if (isOffline.value) {
    console.log('mode offline: activé')
    stopSync()
  } else {
    console.log('mode online: activé')
    startSync()
  }
}

const stopSync = () => {
  if (liveSync) {
    liveSync.cancel()
    liveSync = null
    console.log('Synchronisation live arrêtée')
  }
}

const startSync = () => {
  
  liveSync = PouchDB.sync(storage.value, 'http://leahaberli:20203Marie223@localhost:5984/infradon_projetlibre', {
    live: true,
    retry: true,
  })
    .on('change', (info) => {
      console.log('Changement synchronisé :', info)
      fetchData()
    })
    .on('error', (err) => {
      console.error('Erreur sync :', err)
    })

  console.log('Synchronisation live relancée')
}

</script>

<template>
  <button
    @click="toggleOffline"
    :style="{
      marginBottom: '10px',
      backgroundColor: isOffline ? 'red' : 'green',
      color: 'white',
      border: 'none',
      padding: '8px 16px',
      cursor: 'pointer',
      borderRadius: '4px',
    }"
  >
    {{ isOffline ? 'mode offline' : 'mode online' }}
  </button>
  <h1>Fetch Data</h1>
  <input v-model="searchQuery" @input="indexNumber" placeholder="Rechercher par numéro..." />
  <button @click="createDoc" style="margin: 10px">Ajouter un document</button>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.title }}</h2>
    <p>{{ post.content }}</p>
    <button @click="deleteDoc(post)">Supprimer le document</button><br />
    <button @click="updateDoc(post)">Modifier le document</button>
  </article>
</template>
