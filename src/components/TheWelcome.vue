<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind)

interface Comment {
  _id: string
  _rev?: string
  _conflicts?: string[]
  post_id: string
  comment_content: string
}

declare interface Post {
  _id: string
  _rev: string
  _conflicts?: string[]
  post_name: string
  post_content: string
  post_likes: number
  post_comments: string[]
  attributes: {
    post_category: string
    creation_date: any
  }
  comments?: Comment[]
}

const url = 'http://admin_loann:2B$8a#oq7z89E9#g@localhost:5984/infradonn2'
const opts = { live: true, retry: true }

// Référence à la base de données
const storage = ref()
// Données stockées
const postsData = ref<Post[]>([])
const syncManager = ref<any>(null)

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  // generateRandomPosts(12)
})

const initDatabase = () => {
  console.log('=> Connexion à la base de données.')
  const dbLocal = new PouchDB('local_collection')
  console.log('=> Connecté à la collection : ' + dbLocal.name)
  if (dbLocal) {
    storage.value = dbLocal
    initIndex(dbLocal)
    dbLocal.replicate
      .from(url)
      .on('complete', syncData)
      .then((_result) => {
        fetchData()
      })
  } else {
    console.error('Something went wrong.', Error)
  }
}

const onPaused = () => {
  console.error('Paused')
}

const onError = () => {
  console.error('Erreur')
}
/*
// FACTORY donné par IA
const generateRandomPosts = async (count: number) => {
  const posts = []

  for (let i = 0; i < count; i++) {
    posts.push({
      _id: `${Date.now()}_${i}`,
      post_name: `${Math.random().toString(36).substring(2)}`,
      post_content: `${Math.random().toString(36).substring(2)}`,
      post_likes: `${Math.round(Math.random() * 10)}`,
      attributes: {
        post_category: ['videogames', 'reading', 'cooking'][Math.floor(Math.random() * 3)],
        creation_date: new Date(),
      },
    })
  }
  if (storage.value) {
    await storage.value.bulkDocs(posts)
    console.log(`${count} documents générés`)
    fetchData()
  }
}
*/
// Récupération des données
const fetchData = () => {
  storage.value
    .find({
      selector: {
        'attributes.post_category': indexCategory.value,
        post_likes: { $gte: 0 },
      },
      sort: [{ 'attributes.post_category': 'asc' }, { post_likes: 'desc' }],
      limit: 5,
    })
    .then((result: any) => {
      postsData.value = result.docs
      result.docs.forEach((post: Post) => fetchComments(post._id))
    })
    /*
    .then((result: any) => {
      return Promise.all(
        result.docs.map((post: any) => storage.value.get(post._id, { conflicts: true })),
      )
    })
    .then((allPosts: any) => {
      postsData.value = allPosts
    })
      */
    .catch((error: any) => {
      console.error('Erreur lors de la récupération des posts :', error)
    })
}

const fetchComments = (postId: any) => {
  storage.value
    .find({
      selector: {
        post_id: postId,
      },
    })
    .then((result: any) => {
      const post = postsData.value.find((p) => p._id === postId)
      if (post) post.comments = result.docs
    })
}

const initIndex = (db: any) => {
  db.createIndex({
    index: { fields: ['attributes.post_category', 'post_likes'] },
  })
    .then(function (_result: any) {
      console.log('Index créés')
    })
    .catch((error: any) => {
      console.error("Erreur dans la création de l'index", error)
    })
}

const logInLogOut = () => {
  if (syncManager.value) {
    stopSync()
  } else {
    syncData()
  }
}

const syncData = () => {
  if (syncManager.value) {
    console.log('Synchro déjà établie.')
    return
  } else {
    syncManager.value = storage.value
      .sync(url, opts)
      .on('change', fetchData)
      .on('paused', onPaused)
      .on('error', onError)
    console.log('Synchro commencée.')
  }
}

const stopSync = () => {
  if (!syncManager.value) {
    console.error('Pas de synchro active.')
  } else {
    syncManager.value.cancel()
    syncManager.value = null
    console.log('Synchronisation arrêtée')
  }
}

const postTitle = ref('')
const postContent = ref('')
const postCategory = ref('videogames')
const indexCategory = ref('videogames')

const addDoc = (title: any, content: any, category: any) => {
  storage.value
    .post({
      post_name: title,
      post_content: content,
      attributes: {
        post_category: category,
        creation_date: new Date(),
      },
    })
    .then(function (response: any) {
      console.log(response)
      fetchData()
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const updateDoc = (post: Post) => {
  storage.value
    .put({
      _id: post._id,
      _rev: post._rev,
      post_name: post.post_name,
      post_content: post.post_content,
      post_likes: post.post_likes,
      attributes: {
        post_category: post.attributes.post_category,
        creation_date: new Date(),
      },
    })
    .then(function (response: any) {
      console.log(response)
      fetchData()
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const removeDoc = (post: Post) => {
  storage.value
    .remove({
      _id: post._id,
      _rev: post._rev,
      post_name: post.post_name,
      post_content: post.post_content,
      post_likes: post.post_likes,
      attributes: {
        post_category: post.attributes.post_category,
        creation_date: post.attributes.creation_date,
      },
    })
    .then(function (response: any) {
      console.log(response)
      fetchData()
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const addLike = (post: Post) => {
  post.post_likes++
  updateDoc(post)
}

const addComment = () => {}
</script>
<template>
  <h1>Fetch Data</h1>
  <label>Changer catégorie </label>
  <select v-model="indexCategory" @change="fetchData()">
    <option value="videogames">Jeux vidéo</option>
    <option value="reading">Lire</option>
    <option value="cooking">Cuisiner</option>
  </select>
  <div class="flex">
    <p>Online :</p>
    <label class="switch">
      <input @click="logInLogOut()" type="checkbox" checked />
      <span class="slider round"></span>
    </label>

    <!--  <button @click="syncData()">Sync Database</button>-->
  </div>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <form id="updateDoc" name="updateDoc" @submit.prevent="updateDoc(post)">
      <input
        class="message"
        type="text"
        id="updatePostTitle"
        v-model="post.post_name"
        name="updatePostTitle"
        :placeholder="post.post_name"
        required
        minlength="1"
      />
      <input
        class="message"
        type="text"
        id="updatePostContent"
        v-model="post.post_content"
        name="updatePostContent"
        :placeholder="post.post_content"
        required
        minlength="1"
      />
      <span style="color: #fff">{{ post.post_likes }} ❤️</span>
      <button @click="addLike(post)" type="button">Add like</button>
      <select
        id="postCategory"
        v-model="post.attributes.post_category"
        name="postCategory"
        required
      >
        <option value="videogames">Jeux vidéo</option>
        <option value="reading">Lire</option>
        <option value="cooking">Cuisiner</option>
      </select>
      <button type="submit">Update</button>
      <button @click="removeDoc(post)">Delete</button>
      <span v-for="comment in post.comments" :key="comment._id">
        {{ comment.comment_content }}
      </span>
      <button @click="addComment()">Add comment</button>
      <span class="conflicts" v-if="post._conflicts">Conflits détectés</span>
    </form>
  </article>
  <br /><br />
  <form id="addDoc" name="addDoc" @submit.prevent="addDoc(postTitle, postContent, postCategory)">
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
    /><br />
    <select v-model="postCategory" required>
      <option value="videogames">Jeux vidéo</option>
      <option value="reading">Lire</option>
      <option value="cooking">Cuisiner</option>
    </select>
    <button type="submit">Create</button>
  </form>
</template>
<style scoped>
.conflicts {
  color: #ff0000;
}
.flex {
  display: flex;
  width: 100%;
  height: auto;
}
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
/*
input.message {
  display: flex;
  background-color: transparent;
  box-shadow: none;
  color: white;
}
*/

.switch {
  position: relative;
  display: inline-block;
  width: 60px;
  height: 34px;
}

.switch input {
  opacity: 0;
  width: 0;
  height: 0;
}

.slider {
  position: absolute;
  cursor: pointer;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: #ccc;
  -webkit-transition: 0.4s;
  transition: 0.4s;
}

.slider:before {
  position: absolute;
  content: '';
  height: 26px;
  width: 26px;
  left: 4px;
  bottom: 4px;
  background-color: white;
  -webkit-transition: 0.4s;
  transition: 0.4s;
}

input:checked + .slider {
  background-color: #2196f3;
}

input:focus + .slider {
  box-shadow: 0 0 1px #2196f3;
}

input:checked + .slider:before {
  -webkit-transform: translateX(26px);
  -ms-transform: translateX(26px);
  transform: translateX(26px);
}

/* Rounded sliders */
.slider.round {
  border-radius: 34px;
}

.slider.round:before {
  border-radius: 50%;
}
</style>
