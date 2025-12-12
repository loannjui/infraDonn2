<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
import '../assets/main.css'
import '../assets/base.css'
import '../assets/custom.css'

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
  attributes: {
    post_category: string
    creation_date: any
  }
  comments?: Comment[]
}

const urlPosts = 'http://admin_loann:2B$8a#oq7z89E9#g@localhost:5984/infradonn2'
const urlComments = 'http://admin_loann:2B$8a#oq7z89E9#g@localhost:5984/infradonn2-comments'
const opts = { live: true, retry: true }
const changeOpts = { since: 'now', live: true, include_docs: true }

// Référence à la base de données
const postsDB = ref()
const commentsDB = ref<any>(null)

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
  const dbComments = new PouchDB('local_comments')
  console.log('=> Connecté à la collection : ' + dbComments.name)
  if (dbLocal) {
    postsDB.value = dbLocal
    commentsDB.value = dbComments
    initIndex(dbLocal)
    // DB POSTS
    dbLocal
      .changes(changeOpts)
      .on('change', (change) => {
        console.log(change)
        fetchData()
      })
      .on('error', (err) => {
        console.error(err)
      })
    // DB COMMENTS
    dbComments
      .changes(changeOpts)
      .on('change', (change) => {
        console.log(change)
        fetchData()
      })
      .on('error', (err) => {
        console.error(err)
      })

    dbLocal.replicate
      .from(urlPosts)
      .on('complete', syncData)
      .then((_result) => {
        fetchData()
      })
  } else {
    console.error('Something went wrong.', Error)
  }
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
  if (postsDB.value) {
    await postsDB.value.bulkDocs(posts)
    console.log(`${count} documents générés`)
    fetchData()
  }
}
*/
// Récupération des posts
const fetchData = () => {
  postsDB.value
    .find({
      selector: {
        'attributes.post_category': indexCategory.value,
        post_likes: { $gte: 0 },
      },
      sort: [{ post_likes: 'desc' }],
      limit: 10,
    })
    .then((result: any) => {
      postsData.value = result.docs
      result.docs.forEach((post: Post) => fetchComments(post._id))
    })
    .catch((error: any) => {
      console.error('Erreur lors de la récupération des posts :', error)
    })
}
// Récupération des comments
const fetchComments = (postId: any) => {
  commentsDB.value
    .find({
      selector: { post_id: postId },
    })
    .then((result: any) => {
      const post = postsData.value.find((p) => p._id === postId)
      if (post) post.comments = result.docs
    })
}

const initIndex = (db: any) => {
  // Tri par catégorie
  db.createIndex({
    index: { fields: ['attributes.post_category'] },
  })
    .then(function (_result: any) {
      console.log('Index catégories créé')
    })
    .catch((error: any) => {
      console.error("Erreur dans la création de l'index", error)
    })

  // Tri par Likes
  db.createIndex({
    index: { fields: ['post_likes'] },
  })
    .then((_result: any) => {
      console.log('Index pour post_likes créé')
    })
    .catch((error: any) => {
      console.error("Erreur dans la création de l'index post_likes", error)
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
    syncManager.value = {
      posts: postsDB.value
        .sync(urlPosts, opts)
        .on('change', (_info: any) => {
          fetchData()
        })
        .on('paused', (_info: any) => {
          fetchData()
        })
        .on('active', () => {
          console.log('Synchro post active')
        })
        .on('error', (error: any) => {
          console.error(error)
        }),

      comments: commentsDB.value
        .sync(urlComments, opts)
        .on('change', (_info: any) => {
          fetchData()
        })
        .on('paused', () => {
          fetchData()
        })
        .on('active', () => {
          console.log('Synchro commentaires active')
        })
        .on('error', (error: any) => {
          console.error(error)
        }),
    }
    console.log('Synchro commencée.')

    fetchData()
  }
}

const stopSync = () => {
  if (!syncManager.value) {
    console.error('Pas de synchro active.')
  } else {
    // SYNCHRO POSTS
    if (syncManager.value.posts) {
      syncManager.value.posts.cancel()
    }
    // SYNCHRO COMMENTS
    if (syncManager.value.comments) {
      syncManager.value.comments.cancel()
    }
    syncManager.value = null
    console.log('Synchronisation arrêtée')
  }
}

const postTitle = ref('')
const postContent = ref('')
const postCategory = ref('videogames')
const indexCategory = ref('videogames')

const commentContent = ref([])

const addDoc = (title: any, content: any, category: any) => {
  postsDB.value
    .post({
      post_name: title,
      post_content: content,
      post_likes: 0,
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
  postsDB.value
    .put({
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
    .then(function (_response: any) {
      fetchData()
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const removeDoc = (post: Post) => {
  postsDB.value
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

const addComment = (postId: any, commentContent: any) => {
  commentsDB.value
    .post({
      post_id: postId,
      comment_content: commentContent,
    })
    .then(function (response: any) {
      console.log(response)
      fetchData()
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const updateComment = (comment: Comment) => {
  commentsDB.value
    .put({
      _id: comment._id,
      _rev: comment._rev,
      post_id: comment.post_id,
      comment_content: comment.comment_content,
    })
    .then(function (_response: any) {
      fetchData()
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const removeComment = (comment: Comment) => {
  commentsDB.value
    .remove({
      _id: comment._id,
      _rev: comment._rev,
      post_id: comment.post_id,
      comment_content: comment.comment_content,
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
  <label>Changer catégorie </label>
  <select v-model="indexCategory" @change="fetchData()">
    <option value="videogames">Jeux vidéo</option>
    <option value="reading">Lire</option>
    <option value="cooking">Cuisiner</option>
  </select>

  <p>Online :</p>
  <label class="switch">
    <input @click="logInLogOut()" type="checkbox" checked />
    <span class="slider round"></span>
  </label>

  <!--  <button @click="syncData()">Sync Database</button>-->
  <div class="flex">
    <article v-for="(post, index) in postsData" :key="post._id">
      <form name="updateDoc" @submit.prevent="updateDoc(post)">
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
        <br />
        <label><h2>Commentaire(s)</h2></label>
        <div v-for="comment in post.comments" :key="comment._id">
          <input
            type="text"
            id="updateCommentContent"
            name="updateCommentContent"
            :placeholder="comment.comment_content"
            required
            minlength="1"
            v-model="comment.comment_content"
          />

          <button type="button" @click="updateComment(comment)">Update Com</button>
          <button type="button" @click="removeComment(comment)">Remove Com</button>
        </div>
        <br />
        <h2>Nouveau commentaire</h2>
        <input
          class="comment"
          type="text"
          v-model="commentContent[index]"
          name="addComment"
          placeholder="Votre commentaire"
          minlength="1"
        />
        <button @click="addComment(post._id, commentContent[index])">Add comment</button>

        <span class="conflicts" v-if="post._conflicts">Conflits détectés</span>
      </form>
    </article>
  </div>
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
