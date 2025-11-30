<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

import FindPlugin from 'pouchdb-find'
PouchDB.plugin(FindPlugin)

interface Comment {
  id: string
  author: string
  text: string
}

interface Message {
  _id?: string
  _rev?: string
  type: 'message'
  title: string
  content: string
  comments: Comment[]
  likes: number
  showAllComments?: boolean
}

const storage = ref<any>(null)
const postsData = ref<Message[]>([])
const displayedMessages = ref<Message[]>([])
const isOffline = ref(false)
let liveSync: any = null

const newCommentText = ref('')
const newMessageContent = ref('')
const searchQuery = ref('')
const sortByLikes = ref(false)
const showTop10 = ref(false)

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchMessages()
  console.log(postsData.value)
  startSync()
  storage.value.createIndex({
    index: { fields: ['content'] },
  })
})

const initDatabase = () => {
  storage.value = new PouchDB('infradon_local')
}

const fetchMessages = () => {
  return storage.value.allDocs({ include_docs: true })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.rows)

      let list = result.rows
        .map((r: any) => r.doc)
        .filter((doc: any) => doc && doc.type === 'message')

      if (sortByLikes.value || showTop10.value) {
        list = list.sort((a: any, b: any) => (b.likes || 0) - (a.likes || 0))
      }

      list.forEach((msg: Message) => {
        if (msg.showAllComments === undefined) msg.showAllComments = false
      })

      postsData.value = list
      displayedMessages.value = showTop10.value ? list.slice(0, 10) : list
    })
    .catch((error: any) => {
      console.error('Erreur lors de la récupération des données :', error)
    })
}


const createMessage = () => {
  if (!newMessageContent.value.trim()) return

  const msg: Message = {
    _id: 'msg_' + Date.now(),
    type: 'message',
    title: 'Nouveau message',
    content: newMessageContent.value,
    comments: [],
    likes: 0,
  }

  return storage.value.put(msg).then((response :any) => {
    newMessageContent.value = ''
    fetchMessages()
    console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const updateMessage = (msg: Message) => {
  const newQuote = prompt('Modification de la citation :')
  if (!newQuote) return

  storage.value
    .get(msg._id)
    .then((fresh: any) => {
      fresh.content = newQuote
      return storage.value.put(fresh)
    })
    .then((response: any) => {
      fetchMessages()
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const deleteMessage = (msg: Message) => {
  return storage.value.remove(msg).then((response: any) => {
      fetchMessages()
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const likeMessage = (msg: Message) => {
  storage.value
    .get(msg._id)
    .then((fresh: any) => {
      fresh.likes = (fresh.likes || 0) + 1
      return storage.value.put(fresh)
    })
    .then(fetchMessages)
}

const searchMessages = () => {
  if (!searchQuery.value.trim()) return fetchMessages()

  storage.value
    .find({
      selector: {
        content: { $regex: searchQuery.value },
      },
    })
    .then((res: any) => {
      postsData.value = res.docs
      displayedMessages.value = showTop10.value ? res.docs.slice(0, 10) : res.docs
    })
}

const addComment = (msg: Message, text: string, author = randomItem(randomNames)) => {
  if (!text.trim()) return

  storage.value
    .get(msg._id)
    .then((fresh: any) => {
      fresh.comments.push({
        id: 'c_' + Date.now(),
        author,
        text,
      })
      return storage.value.put(fresh)
    })
    .then(() => {
      newCommentText.value = ''
      fetchMessages()
    })
}

const updateComment = (msg: Message, commentId: string) => {
  const newText = prompt('Nouveau commentaire :')
  if (!newText) return

  storage.value
    .get(msg._id)
    .then((fresh: any) => {
      const comment = fresh.comments.find((c: any) => c.id === commentId)
      if (!comment) return
      comment.text = newText
      return storage.value.put(fresh)
    })
    .then(fetchMessages)
}

const deleteComment = (msg: Message, commentId: string) => {
  storage.value
    .get(msg._id)
    .then((fresh: any) => {
      fresh.comments = fresh.comments.filter((c: any) => c.id !== commentId)
      return storage.value.put(fresh)
    })
    .then(fetchMessages)
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
  liveSync = PouchDB.sync(
    storage.value,
    'http://leahaberli:20203Marie223@localhost:5984/infradon_projetlibre',
    {
      live: true,
      retry: true,
    },
  )
    .on('change', (info) => {
      console.log('Changement synchronisé :', info)
      fetchMessages()
    })
    .on('error', (err) => {
      console.error('Erreur sync :', err)
    })

    console.log('Synchronisation live relancée')
}

const randomNames = ['Colette', 'Emile', 'Delphine', 'Samuel', 'Jordan', 'Chloé', 'Benoît', 'Inoé', 'Marike']
const randomQuotes = [
  'La vie est un écho : ce que tu envoies revient toujours.',
  'Le courage ne se mesure pas à l’absence de peur, mais à la volonté d’avancer malgré elle.',
  'Chaque petit geste compte, car c’est l’ensemble qui crée le monde.',
  'La patience est le fil invisible qui tisse les plus beaux succès.',
  'Parfois, se perdre est la seule façon de se retrouver.',
  'Le bonheur n’est pas un lieu, mais une manière de voyager.',
  'Ce n’est pas la vitesse qui compte, mais la direction.',
]
const randomComments = [
  'Très inspirant !',
  'J’adore cette citation.',
  'Ça me fait réfléchir.',
  'Merci pour ce partage.',
  'C’est exactement ce dont j’avais besoin aujourd’hui.',
]

function randomItem(arr: string[]): string {
  return arr[Math.floor(Math.random() * arr.length)] ?? ''
}

const generateFakeMessages = (count: number) => {
  let chain = Promise.resolve()

  for (let i = 0; i < count; i++) {
    const msg: Message = {
      _id: 'quote' + i,
      type: 'message',
      title: 'Citation #' + (i + 1),
      content: randomItem(randomQuotes),
      comments: [],
      likes: Math.floor(Math.random() * 100),
    }

    const nbComments = Math.floor(Math.random() * 5)
    for (let c = 0; c < nbComments; c++) {
      msg.comments.push({
        id: 'com' + c,
        author: randomItem(randomNames),
        text: randomItem(randomComments),
      })
    }

    chain = chain.then(() => storage.value.put(msg))
  }

  chain.then(() => {
    console.log(count + ' messages générés !')
    fetchMessages()
  })
}
</script>

<template>
  <button
    @click="toggleOffline"
    :style="{
      marginBottom: '10px',
      marginTop: '10px',
      backgroundColor: isOffline ? 'red' : 'green',
      color: 'white',
      border: 'none',
      padding: '8px 16px',
      cursor: 'pointer',
      borderRadius: '4px',
    }"
  >
    {{ isOffline ? 'Mode Offline' : 'Mode Online' }}
  </button>

  <h1 style="color: blueviolet; font-weight: 600">Gestionnaire de citations</h1>

  <input
    v-model="newMessageContent"
    placeholder="Écrivez votre citation...."
    style="width: 100%; min-height: 40px; margin-top: 10px; border-radius: 4px;"
  />

  <button @click="createMessage" style="margin-top: 10px; background-color: blueviolet; color: white; padding: 9px; border-radius: 8px; border: 0px;">Publier la citation</button>

  <button @click="generateFakeMessages(20)" style="margin-left: 20px; margin-top: 10px; background-color: black; color: blueviolet; padding: 9px; border-radius: 8px; border: 0px;">Générer 20 citations</button>

  <input
    v-model="searchQuery"
    @input="searchMessages"
    placeholder="Rechercher une citation...."
    style="margin-top: 30px; margin-bottom: 10px; width: 100%"
  />

  <label style="color: white">
    <input type="checkbox" v-model="sortByLikes" @change="fetchMessages" />
    Trier par nombre de likes
  </label>

  <button
    @click="
      showTop10 = !showTop10,
      fetchMessages()
    "
    style="margin-left: 10px; background-color: blueviolet; color: white; padding: 5px; border-radius: 8px; border: 0px;"
  >
    {{ showTop10 ? 'Afficher toutes les citations' : 'Afficher Top 10 des plus likés' }}
  </button>

  <article
    v-for="msg in displayedMessages"
    :key="msg._id"
    style="margin-top: 20px; padding: 10px; border: 1px solid blueviolet; border-radius: 6px "
  >
    <h2 style="color: white;">{{ msg.content }}</h2>

    <button @click="likeMessage(msg)" style="margin-top: 5px; margin-right: 5px; color: white; background-color: blueviolet; padding: 5px; border-radius: 8px; border: 0px;">Like ({{ msg.likes || 0 }})</button>
    <button @click="updateMessage(msg)" style="margin-top: 5px; margin-right: 5px; color: blueviolet; background-color: black; padding: 5px; border-radius: 8px; border: 0px;">Modifier</button>
    <button @click="deleteMessage(msg)" style="margin-top: 5px; margin-right: 5px; color: blueviolet; background-color: black; padding: 5px; border-radius: 8px; border: 0px;">Supprimer</button>

    <h3>Commentaires :</h3>

    <div
      v-for="c in msg.showAllComments ? msg.comments : msg.comments.slice(0, 1)"
      :key="c.id"
      style="background: #f3f3f3; padding: 8px; margin: 4px; border-radius: 4px; color: black"
    >
      <strong>{{ c.author }}</strong> : {{ c.text }}
      <br />
      <button @click="updateComment(msg, c.id)" style="margin-top: 5px; margin-right: 5px; color: blueviolet; padding: 5px; border-radius: 8px; border: 0px;">Modifier</button>
      <button @click="deleteComment(msg, c.id)" style="margin-top: 5px; margin-right: 5px; color: blueviolet; padding: 5px; border-radius: 8px; border: 0px;">Supprimer</button>
    </div>

    <button v-if="msg.comments.length > 1" @click="msg.showAllComments = !msg.showAllComments" style="margin-top: 5px; margin-right: 5px; color: black; padding: 5px; border-radius: 8px; border: 0px;">
      {{ msg.showAllComments ? 'Réduire' : 'Voir tous les commentaires' }}
    </button>

    <input
      v-model="newCommentText"
      placeholder="Ajouter un commentaire..."
      style="margin-top: 10px; border-radius: 4px;"
    />
    <button @click="addComment(msg, newCommentText)" style="margin-top: 5px; margin-right: 5px; color: white; background-color: blueviolet; padding: 5px; border-radius: 8px; border: 0px;">Envoyer</button>
  </article>
</template>
