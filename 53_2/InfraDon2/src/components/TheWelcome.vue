<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

import FindPlugin from 'pouchdb-find'
PouchDB.plugin(FindPlugin)

interface Comment {
  _id?: string
  _rev?: string
  type: 'comment'
  messageId: string
  author: string
  text: string
  creation_date?: string
}

interface Message {
  _id?: string
  _rev?: string
  type: 'message'
  title?: string
  content: string
  likes?: number
  creation_date?: string
  comments?: Comment[]
  showAllComments?: boolean
  newCommentText?: string
}

const messagesDB = ref<any>(null) 
const commentsDB = ref<any>(null) 
const postsData = ref<Message[]>([])
const displayedMessages = ref<Message[]>([])
const isOffline = ref(false)

const newCommentText = ref('')
const newMessageContent = ref('')
const searchQuery = ref('')
const sortByLikes = ref(false)
const showTop10 = ref(false)

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  createIndexes()
    .then(() => fetchMessages())
    .then(() => startSync())
    .catch((err: any) => console.error('Erreur init:', err))
  console.log(postsData.value)
})

const initDatabase = () => {
  messagesDB.value = new PouchDB('infradon_messages')
  commentsDB.value = new PouchDB('infradon_comments')
}

const createIndexes = () => {
  if (!messagesDB.value || !commentsDB.value) return Promise.reject('DB not initialized')

  return messagesDB.value
    .createIndex({ index: { fields: ['type', 'likes'] } })
    .then(() => messagesDB.value.createIndex({ index: { fields: ['type', 'content'] } }))
    .then(() => commentsDB.value.createIndex({ index: { fields: ['type', 'messageId'] } }))
    .then(() => commentsDB.value.createIndex({ index: { fields: ['creation_date'] } }))
    .catch((err: any) => console.warn('createIndexes warning:', err))
}


const fetchMessages = () => {
  if (!messagesDB.value) return Promise.resolve()

  const finalizeDisplayed = (msgs: Message[]) => {
    postsData.value = msgs
    postsData.value.forEach((m) => {
      if (m.showAllComments === undefined) m.showAllComments = false
      if (!m.comments) m.comments = []
    })
    displayedMessages.value = postsData.value
    return Promise.all(
      displayedMessages.value.map((m: any) => {
        return commentsDB.value
          .find({
            selector: { type: 'comment', messageId: m._id },
            limit: 1,
          })
          .then((cres: any) => {
            m.comments = cres.docs?.length ? (cres.docs as Comment[]) : []
          })
          .catch(() => {
            m.comments = []
          })
      }),
    )
  }

  const query: any = { selector: { type: 'message' } }

  if (searchQuery.value && searchQuery.value.trim()) {
    query.selector.content = { $regex: searchQuery.value }
  }

  if (sortByLikes.value) {
    query.sort = [{ type: 'asc' }, { likes: 'desc' }]
  }

  if (showTop10.value) {
    query.limit = 10
  }

  return messagesDB.value
    .find(query)
    .then((res: any) => {
      res.docs.forEach((m: Message) => {
        if (m.likes === undefined) m.likes = 0
      })
      return finalizeDisplayed(res.docs)
    })
    .catch((err: any) => console.error('Erreur fetchMessages:', err))
}


const createMessage = () => {
  const id = 'msg_' + Date.now() + '_' + Math.floor(Math.random() * 1000)
  const msg: Message = {
    _id: id,
    type: 'message',
    title: 'Nouveau message',
    content: newMessageContent.value,
    likes: 0,
    creation_date: new Date().toISOString(),
  }

  return messagesDB.value
    .put(msg)
    .then(() => {
      newMessageContent.value = ''
      return fetchMessages()
    })
    .catch((err: any) => console.error('Erreur createMessage:', err))
}

const updateMessage = (msg: Message) => {
  const newQuote = prompt('Modification de la citation :', msg.content)
  if (!newQuote) return

  return messagesDB.value
    .get(msg._id)
    .then((fresh: any) => {
      fresh.content = newQuote
      return messagesDB.value.put(fresh)
    })
    .then(() => fetchMessages())
    .catch((err: any) => console.error('Erreur updateMessage:', err))
}

const deleteMessage = (msg: Message) => {
  return commentsDB.value
    .find({ selector: { type: 'comment', messageId: msg._id } })
    .then((cres: any) => {
      const toDelete = (cres.docs || []).map((c: any) => ({ ...c, _deleted: true }))
      if (toDelete.length) {
        return commentsDB.value.bulkDocs(toDelete)
      } else {
        return Promise.resolve()
      }
    })
    .then(() => messagesDB.value.get(msg._id))
    .then((m: any) => messagesDB.value.remove(m))
    .then(() => fetchMessages())
    .catch((err: any) => console.error('Erreur deleteMessage:', err))
}

const likeMessage = (msg: Message) => {
  return messagesDB.value
    .get(msg._id)
    .then((fresh: any) => {
      fresh.likes = (fresh.likes || 0) + 1
      return messagesDB.value.put(fresh)
    })
    .then(() => fetchMessages())
    .catch((err: any) => console.error('Erreur likeMessage:', err))
}

const addComment = (msg: Message, text: string, author = randomItem(randomNames)) => {
  if (!text || !text.trim()) return Promise.resolve()

  const id = 'cmt_' + Date.now() + '_' + Math.floor(Math.random() * 1000)
  const doc: Comment = {
    _id: id,
    type: 'comment',
    messageId: msg._id as string,
    author,
    text,
    creation_date: new Date().toISOString(),
  }

  return commentsDB.value
    .put(doc)
    .then(() => {
      newCommentText.value = ''
      if (msg.showAllComments) return loadCommentsForMessage(msg)
      else return loadFirstCommentForMessage(msg)
    })
    .catch((err: any) => console.error('Erreur addComment:', err))
}

const updateComment = (commentId: string, msg?: Message) => {
  const newText = prompt('Modifier le commentaire :')
  if (!newText) return

  return commentsDB.value
    .get(commentId)
    .then((fresh: any) => {
      fresh.text = newText
      return commentsDB.value.put(fresh)
    })
    .then(() => {
      if (msg) {
        return msg.showAllComments ? loadCommentsForMessage(msg) : loadFirstCommentForMessage(msg)
      } else {
        return fetchMessages()
      }
    })
    .catch((err: any) => console.error('Erreur updateComment:', err))
}

const deleteComment = (commentId: string, msg?: Message) => {
  return commentsDB.value
    .get(commentId)
    .then((fresh: any) => commentsDB.value.remove(fresh))
    .then(() => {
      if (msg) {
        return msg.showAllComments ? loadCommentsForMessage(msg) : loadFirstCommentForMessage(msg)
      } else {
        return fetchMessages()
      }
    })
    .catch((err: any) => console.error('Erreur deleteComment:', err))
}

const loadCommentsForMessage = (msg: Message) => {
  return commentsDB.value
    .find({
      selector: { type: 'comment', messageId: msg._id },
      sort: [{ creation_date: 'asc' }],
    })
    .then((res: any) => {
      msg.comments = res.docs as Comment[]
    })
    .catch(() => {
      return commentsDB.value
        .find({ selector: { type: 'comment', messageId: msg._id } })
        .then((r2: any) => {
          msg.comments = r2.docs as Comment[]
        })
        .catch(() => {
          msg.comments = []
        })
    })
}

const loadFirstCommentForMessage = (msg: Message) => {
  return commentsDB.value
    .find({
      selector: { type: 'comment', messageId: msg._id },
      sort: [{ creation_date: 'asc' }],
      limit: 1,
    })
    .then((res: any) => {
      msg.comments = res.docs && res.docs.length ? (res.docs as Comment[]) : []
    })
    .catch(() => {
      return commentsDB.value
        .find({ selector: { type: 'comment', messageId: msg._id }, limit: 1 })
        .then((r2: any) => {
          msg.comments = r2.docs && r2.docs.length ? (r2.docs as Comment[]) : []
        })
        .catch(() => {
          msg.comments = []
        })
    })
}

let syncMessagesHandler: any = null
let syncCommentsHandler: any = null

const startSync = () => {
  if (!messagesDB.value || !commentsDB.value) return
  syncMessagesHandler = PouchDB.sync(
    messagesDB.value,
    'http://leahaberli:20203Marie223@localhost:5984/infradon_messages',
    { live: true, retry: true },
  )
    .on('change', () => fetchMessages())
    .on('error', (err: any) => console.error('Erreur de synchronisation:', err))

  syncCommentsHandler = PouchDB.sync(
    commentsDB.value,
    'http://leahaberli:20203Marie223@localhost:5984/infradon_comments',
    { live: true, retry: true },
  )
    .on('change', () => {
      Promise.all(displayedMessages.value.map((m) => loadFirstCommentForMessage(m))).catch(() => {})
    })
    .on('error', (err: any) => console.error('Sync comments error:', err))
}

const stopSync = () => {
  if (syncMessagesHandler) {
    syncMessagesHandler.cancel()
    syncMessagesHandler = null
  }
  if (syncCommentsHandler) {
    syncCommentsHandler.cancel()
    syncCommentsHandler = null
  }
}

const toggleOffline = () => {
  isOffline.value = !isOffline.value
  if (isOffline.value) stopSync()
  else startSync()
}

const randomNames = [
  'Colette',
  'Emile',
  'Delphine',
  'Samuel',
  'Jordan',
  'Chloé',
  'Benoît',
  'Inoé',
  'Marike',
]
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
  const msgs: Message[] = []
  const cmts: Comment[] = []

  for (let i = 0; i < count; i++) {
    const idMsg = 'msg_' + Date.now() + '_' + i + '_' + Math.floor(Math.random() * 1000)
    const m: Message = {
      _id: idMsg,
      type: 'message',
      title: 'Citation #' + (i + 1),
      content: randomItem(randomQuotes),
      likes: Math.floor(Math.random() * 100),
      creation_date: new Date().toISOString(),
    }
    msgs.push(m)

    const nb = Math.floor(Math.random() * 5)
    for (let c = 0; c < nb; c++) {
      const idC = 'cmt_' + Date.now() + '_' + i + '_' + c + '_' + Math.floor(Math.random() * 1000)
      cmts.push({
        _id: idC,
        type: 'comment',
        messageId: idMsg,
        author: randomItem(randomNames),
        text: randomItem(randomComments),
        creation_date: new Date().toISOString(),
      })
    }
  }
  return messagesDB.value
    .bulkDocs(msgs)
    .then(() => commentsDB.value.bulkDocs(cmts))
    .then(() => {
      console.log(count + ' messages générés !')
      return fetchMessages()
    })
    .catch((err: any) => console.error('Erreur generateFakeMessages:', err))
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
    style="width: 100%; min-height: 40px; margin-top: 10px; border-radius: 4px"
  />

  <button
    @click="createMessage"
    style="
      margin-top: 10px;
      background-color: blueviolet;
      color: white;
      padding: 9px;
      border-radius: 8px;
      border: 0px;
    "
  >
    Publier la citation
  </button>

  <button
    @click="generateFakeMessages(20)"
    style="
      margin-left: 20px;
      margin-top: 10px;
      background-color: black;
      color: blueviolet;
      padding: 9px;
      border-radius: 8px;
      border: 0px;
    "
  >
    Générer 20 citations
  </button>

  <input
    v-model="searchQuery"
    @input="fetchMessages"
    placeholder="Rechercher une citation...."
    style="margin-top: 30px; margin-bottom: 10px; width: 100%"
  />

  <label style="color: white">
    <input type="checkbox" v-model="sortByLikes" @change="fetchMessages" />
    Trier par nombre de likes
  </label>

  <button
    @click="((showTop10 = !showTop10), fetchMessages())"
    style="
      margin-left: 10px;
      background-color: blueviolet;
      color: white;
      padding: 5px;
      border-radius: 8px;
      border: 0px;
    "
  >
    {{ showTop10 ? 'Afficher toutes les citations' : 'Afficher Top 10 des plus likés' }}
  </button>

  <article
    v-for="msg in displayedMessages"
    :key="msg._id"
    style="margin-top: 20px; padding: 10px; border: 1px solid blueviolet; border-radius: 6px"
  >
    <h2 style="color: white">{{ msg.content }}</h2>

    <button
      @click="likeMessage(msg)"
      style="
        margin-top: 5px;
        margin-right: 5px;
        color: white;
        background-color: blueviolet;
        padding: 5px;
        border-radius: 8px;
        border: 0px;
      "
    >
      Like ({{ msg.likes || 0 }})
    </button>
    <button
      @click="updateMessage(msg)"
      style="
        margin-top: 5px;
        margin-right: 5px;
        color: blueviolet;
        background-color: black;
        padding: 5px;
        border-radius: 8px;
        border: 0px;
      "
    >
      Modifier
    </button>
    <button
      @click="deleteMessage(msg)"
      style="
        margin-top: 5px;
        margin-right: 5px;
        color: blueviolet;
        background-color: black;
        padding: 5px;
        border-radius: 8px;
        border: 0px;
      "
    >
      Supprimer
    </button>

    <h3>Commentaires :</h3>

    <div
      v-for="c in msg.showAllComments ? msg.comments : msg.comments ? msg.comments.slice(0, 1) : []"
      :key="c._id"
      style="background: #f3f3f3; padding: 8px; margin: 4px; border-radius: 4px; color: black"
    >
      <strong>{{ c.author }}</strong> : {{ c.text }}
      <br />
      <button
        @click="updateComment(c._id!, msg)"
        style="
          margin-top: 5px;
          margin-right: 5px;
          color: blueviolet;
          padding: 5px;
          border-radius: 8px;
          border: 0px;
        "
      >
        Modifier
      </button>
      <button
        @click="deleteComment(c._id!, msg)"
        style="
          margin-top: 5px;
          margin-right: 5px;
          color: blueviolet;
          padding: 5px;
          border-radius: 8px;
          border: 0px;
        "
      >
        Supprimer
      </button>
    </div>

    <button
      v-if="msg.comments && msg.comments.length ? msg.comments.length > 1 : false"
      @click="
        msg.showAllComments = !msg.showAllComments;
        if (msg.showAllComments) loadCommentsForMessage(msg)
      "
      style="
        margin-top: 5px;
        margin-right: 5px;
        color: black;
        padding: 5px;
        border-radius: 8px;
        border: 0px;
      "
    >
      {{ msg.showAllComments ? 'Réduire' : 'Voir tous les commentaires' }}
    </button>
    <div>
      <input
        v-model="msg.newCommentText"
        placeholder="Ajouter un commentaire..."
        style="margin-top: 10px; border-radius: 4px"
      />
      <button
        @click="addComment(msg, msg.newCommentText || '')"
        style="
          margin-top: 5px;
          margin-right: 5px;
          color: white;
          background-color: blueviolet;
          padding: 5px;
          border-radius: 8px;
          border: 0px;
        "
      >
        Envoyer
      </button>
    </div>
  </article>
</template>
