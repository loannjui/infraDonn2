<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  post_name: string
  post_content: string
  attributes: {
    creation_date: any
  }
}

// Référence à la base de données
const storage = ref()
// Données stockées
const postsData = ref<Post[]>([])

const initDatabase = () => {
  console.log('=> Connexion à la base de données.')
  const db = new PouchDB('http://admin_loann:2B$8a#oq7z89E9#g@localhost:5984/infradonn2')
  if (db) {
    console.log('Connected to collection : ' + db?.name)
    storage.value = db
  } else {
    console.warn('Something went wrong')
  }
}

// Récupération des données
const fetchData = () => {
  // TODO Récupération des données
  // https://pouchdb.com/api.html#batch_fetch
  // Regarder l'exemple avec function allDocs
  // Remplir le tableau postsData avec les données récupérées
  storage.value
    .allDocs({
      include_docs: true,
      attachments: true,
    })
    .then(function (result: any) {
      console.log('=> Données récupérées :', result.rows)
      postsData.value = result.rows.map((row: any) => row.doc)
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données :', error)
    })
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
})

const postTitle = ref('')
const postContent = ref('')

const addDoc = (title: any, content: any) => {
  storage.value
    .post({
      post_name: title,
      post_content: content,
    })
    .then(function (response: any) {
      console.log(response)
      fetchData()
    })
    .catch(function (err: any) {
      console.log(err)
    })
}
</script>

<template>
  <h1>Fetch Data</h1>
  <form id="addDoc" name="addDoc">
    <label for="sendId">Add New Doc</label><br />
    <input
      type="text"
      id="postTitle"
      v-model="postTitle"
      name="postTitle"
      placeholder="Titre du post"
      required
    /><br />
    <input
      type="text"
      id="postContent"
      v-model="postContent"
      name="postContent"
      placeholder="Contenu du post"
      required
    /><br /><br />
    <input @click="addDoc(postTitle, postContent)" value="Envoyer" />
  </form>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>
  </article>
</template>
