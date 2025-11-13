<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  title: string
  content: string
  name: string
  attributes: {
    creation_date: any
  }
}

const storage = ref()
const postsData = ref<Post[]>([])
let counter = 0

const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://leahaberli:20203Marie223@localhost:5984/infradon_projetlibre')
  if (db) {
    console.log('Connected to collection : ' + db?.name)
    storage.value = db
  } else {
    console.warn('Something went wrong')
  }
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

const replicateDatabase = () => {
  if (!storage.value) return

  const localDB = new PouchDB('infradon_local') // base locale
  const remoteDB = storage.value // base distante

  PouchDB.replicate(remoteDB, localDB)
    .on('complete', (info) => {
      console.log('Réplique terminée :', info)
      fetchData()
    })
    .on('error', (err) => {
      console.error('Erreur lors de la réplication :', err)
    })

  localDB
    .allDocs({ include_docs: true })
    .then((result) => console.log('Docs locaux :', result.rows))
    .catch((err) => console.error(err))
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
  console.log(postsData.value)
  replicateDatabase()
})
</script>

<template>
  <h1>Fetch Data</h1>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.title }}</h2>
    <p>{{ post.content }}</p>
    <button @click="deleteDoc(post)">Supprimer le document</button><br />
    <button @click="updateDoc(post)">Modifier le document</button>
  </article>
  <button @click="createDoc">Ajouter un document</button>
</template>
