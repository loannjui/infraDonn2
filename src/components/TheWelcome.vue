<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  _id: string
  _rev: string
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

const updateDoc = (id: any, rev: any, updatePostTitle: any, updatePostContent: any) => {
  storage.value
    .put({
      _id: id,
      _rev: rev,
      post_name: updatePostTitle,
      post_content: updatePostContent,
    })
    .then(function (response: any) {
      console.log(response)
      fetchData()
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const removeDoc = (post: any) => {
  storage.value
    .remove({
      _id: post._id,
      _rev: post._rev,
      post_name: post.post_name,
      post_content: post.post_content,
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
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>
    <form
      id="updateDoc"
      name="updateDoc"
      @submit.prevent="updateDoc(post._id, post._rev, post.post_name, post.post_content)"
    >
      <input
        type="text"
        id="updatePostTitle"
        v-model="post.post_name"
        name="updatePostTitle"
        :placeholder="post.post_name"
        required
        minlength="1"
      />
      <input
        type="text"
        id="updatePostContent"
        v-model="post.post_content"
        name="updatePostContent"
        :placeholder="post.post_content"
        required
        minlength="1"
      />
      <button type="submit">Update</button>
    </form>
    <button @click="removeDoc(post)" value="Delete">Delete</button>
  </article>
  <br /><br />
  <form id="addDoc" name="addDoc" @submit.prevent="addDoc(postTitle, postContent)">
    <label for="sendId">Add New Doc</label><br />
    <input
      type="text"
      id="postTitle"
      v-model="postTitle"
      name="postTitle"
      placeholder="Titre du post"
      required
      minlength="1"
    /><br />
    <input
      type="text"
      id="postContent"
      v-model="postContent"
      name="postContent"
      placeholder="Contenu du post"
      required
      minlength="1"
    /><br /><br />
    <button type="submit">Envoyer</button>
  </form>
</template>
<style scoped>
input {
  font-size: 1rem;
  font-weight: 600;
  font-family:
    Inter,
    -apple-system,
    BlinkMacSystemFont,
    'Segoe UI',
    Roboto,
    Oxygen,
    Ubuntu,
    Cantarell,
    'Fira Sans',
    'Droid Sans',
    'Helvetica Neue',
    sans-serif;
  background-color: white;
  border: none;
  box-shadow: 0 0 15px rgba(0, 0, 0, 10);
  color: black;
  padding: 0.5rem;
  border-radius: 1rem;
  margin-right: 1rem;
  margin-bottom: 1rem;
}
</style>
