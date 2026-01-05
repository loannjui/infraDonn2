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
  comment_likes: number
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
  _attachments?: {}
  // Pour plus facilement supprimer en ayant un nom pour l'image
  attachmentsURL?: Array<{ url: string; attachmentName: string }>
}

const urlPosts = 'http://admin_loann:2B$8a#oq7z89E9#g@localhost:5984/posts_juillerat_loann'
const urlComments = 'http://admin_loann:2B$8a#oq7z89E9#g@localhost:5984/comments_juillerat_loann'
const opts = { live: true, retry: true }
const changeOpts = { since: 'now', live: true, include_docs: true }

// Référence à la base de données
const postsDB = ref()
const commentsDB = ref<any>(null)

// Données stockées
const postsData = ref<Post[]>([])
const syncManager = ref<any>(null)

// Ref pour envoyer un id de post avec une valeur boolean. Utilisé dans fetchComments
const showAllComments = ref<{ [key: string]: boolean }>({})

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
})

const initDatabase = () => {
  console.log('=> Connexion à la base de données.')
  const dbPosts = new PouchDB('local_collection')
  console.log('=> Connecté à la collection : ' + dbPosts.name)
  const dbComments = new PouchDB('local_comments')
  console.log('=> Connecté à la collection : ' + dbComments.name)
  if (dbPosts) {
    postsDB.value = dbPosts
    commentsDB.value = dbComments
    // Création des index
    initCommentIndex(dbComments)
    initPostIndex(dbPosts)
    // DB POSTS
    dbPosts
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

    dbPosts.replicate
      .from(urlPosts)
      .on('complete', syncData)
      .then((result) => {
        console.log(result)
      })
  } else {
    console.error('Something went wrong.', Error)
  }
}
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
      include_docs: true,
    })
    .then((result: any) => {
      // Pour pas supprimer les attachemntsURL si on like/ajoute un commentaire.
      result.docs.forEach((newPost: Post) => {
        // On vérifie si le post existe
        const existingPost = postsData.value.find((p) => p._id === newPost._id)
        // On copie les attchments du post dans le "nouveau" post qui a été liké/updaté/... sinon on rend un tableau vide.
        if (existingPost?.attachmentsURL) {
          newPost.attachmentsURL = existingPost.attachmentsURL
        } else {
          newPost.attachmentsURL = []
        }
      })

      postsData.value = result.docs

      result.docs.forEach((post: Post) => {
        // On regarde si le post a une valeur true/false (donc si tous les commentaires sont affichés ou pas) et on envoie la limite correspondante.
        const limit = showAllComments.value[post._id] ? 1000 : 1
        fetchComments(post._id, limit)
      })
      result.docs.forEach((post: Post) => fetchAttachments(post))
    })
    .catch((err: any) => {
      console.error('Erreur lors de la récupération des posts :', err)
    })
}
// Récupération des comments
const fetchComments = (postId: any, limit: number) => {
  commentsDB.value
    .find({
      selector: {
        post_id: postId,
        comment_likes: { $gte: 0 },
      },
      sort: [{ comment_likes: 'desc' }],
      limit: limit,
    })
    .then((result: any) => {
      const post = postsData.value.find((p) => p._id === postId)
      if (post) post.comments = result.docs
    })
}

// Toggle des commentaires
const toggleShowAllComments = (postId: string) => {
  showAllComments.value[postId] = !showAllComments.value[postId]

  if (showAllComments.value[postId]) {
    fetchComments(postId, 1000)
  } else {
    fetchComments(postId, 1)
  }
}

// Récupération des attachments
const fetchAttachments = (post: Post) => {
  if (!post._attachments) return
  // On vide le tableau à chaque fetch
  post.attachmentsURL = []

  for (const attName in post._attachments) {
    loadAttachments(post, attName)
  }
}

const initPostIndex = (db: any) => {
  // Tri par catégorie
  db.createIndex({
    index: { fields: ['attributes.post_category'] },
  })
    .then(function (_result: any) {
      console.log('Index catégories créé')
    })
    .catch((err: any) => {
      console.error("Erreur dans la création de l'index", err)
    })

  // Tri par Likes
  db.createIndex({
    index: { fields: ['post_likes'] },
  })
    .then((_result: any) => {
      console.log('Index pour post_likes créé')
    })
    .catch((err: any) => {
      console.error("Erreur dans la création de l'index post_likes", err)
    })
}

// Pour récupérer les commentaires avec le plus de likes.
const initCommentIndex = (db: any) => {
  db.createIndex({
    index: { fields: ['comment_likes'] },
  })
    .then(() => {
      console.log('Index pour comment_likes créé')
    })
    .catch((err: any) => {
      console.error("Erreur dans la création de l'index comment_likes", err)
    })
}

const logInLogOut = () => {
  if (syncManager.value) {
    stopSync()
  } else {
    syncData()
  }
}
// Synchro séparée pour les deux DB.
const syncData = () => {
  if (syncManager.value) {
    console.log('Synchro déjà établie.')
    return
  } else {
    syncManager.value = {
      posts: postsDB.value
        .sync(urlPosts, opts)
        .on('change', (_info: any) => {
          console.log('Posts modifiés')
        })
        .on('paused', (_info: any) => {
          console.log('Synchro posts mise en pause')
        })
        .on('active', (_info: any) => {
          console.log('Synchro posts active')
        })
        .on('error', (err: any) => {
          console.error(err)
        }),

      comments: commentsDB.value
        .sync(urlComments, opts)
        .on('change', (_info: any) => {
          console.log('Commentaires modifiés')
        })
        .on('paused', (_info: any) => {
          console.log('Synchro commentaires mise en pause')
        })
        .on('active', (_info: any) => {
          console.log('Synchro commentaires active')
        })
        .on('error', (err: any) => {
          console.error(err)
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

const addPost = (title: any, content: any, category: any) => {
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
    .then((response: any) => {
      console.log(response)
      postTitle.value = ''
      postContent.value = ''
    })
    .catch((error: any) => {
      console.log(error)
    })
}

const updatePost = (post: Post) => {
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
    .then(function (response: any) {
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const removePost = (post: Post) => {
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
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const addPostLike = (post: Post) => {
  post.post_likes++
  updatePost(post)
}

const addComLike = (comment: Comment) => {
  comment.comment_likes++
  updateComment(comment)
}

const addComment = (postId: any, commentContent: any, index: number) => {
  commentsDB.value
    .post({
      post_id: postId,
      comment_content: commentContent,
      comment_likes: 0,
    })
    .then((response: any) => {
      console.log(response)
      commentContent[index] = ''
    })
    .catch((err: any) => {
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
      comment_likes: comment.comment_likes,
    })
    .then((response: any) => {
      console.log(response)
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
      comment_likes: comment.comment_likes,
    })
    .then((response: any) => {
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const addAttachement = (post: Post, event: Event) => {
  const input = event.target as HTMLInputElement
  if (!input.files) return

  const myFile = input.files[0]

  if (!myFile) return

  postsDB.value
    .putAttachment(post._id, myFile.name, post._rev!, myFile, myFile.type)
    .then((response: any) => {
      post._rev = response.rev
      fetchAttachments(post)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const removeAttachment = (post: Post, attachmentName: string) => {
  postsDB.value
    .removeAttachment(post._id, attachmentName, post._rev)
    .then((response: any) => {
      // On met à jour la révision pour éviter les conflits
      post._rev = response.rev
      fetchAttachments(post)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const loadAttachments = (post: Post, attachmentName: string) => {
  postsDB.value
    .getAttachment(post._id, attachmentName)
    .then((blob: Blob) => {
      const url = URL.createObjectURL(blob)
      // Initialise le tableau s'il n'existe pas déjà
      if (!post.attachmentsURL) post.attachmentsURL = []
      // Ensuite on push l'url + le nom de l'image
      post.attachmentsURL.push({ url, attachmentName })
      // On recrée le tableau avec les mêmes valeurs pour forcer un refresh
      postsData.value = [...postsData.value]
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
      <form name="updatePost" @submit.prevent="updatePost(post)">
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
        <div v-if="post.attachmentsURL">
          <div v-for="attachment in post.attachmentsURL" :key="attachment.attachmentName">
            <img :src="attachment.url" alt="" />
            <button type="button" @click="removeAttachment(post, attachment.attachmentName)">
              Delete
            </button>
          </div>
        </div>
        <span style="color: #fff">{{ post.post_likes }} ❤️</span>
        <button @click="addPostLike(post)" type="button">Like</button>
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
        <button type="submit">Update Post</button>
        <button @click="removePost(post)">Delete Post</button>
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
          <span style="color: #fff">{{ comment.comment_likes }} ❤️</span>
          <button @click="addComLike(comment)" type="button">Like</button>
          <button type="button" @click="updateComment(comment)">Update</button>
          <button type="button" @click="removeComment(comment)">Remove</button>
        </div>
        <button type="button" @click="toggleShowAllComments(post._id)" v-if="post.comments">
          {{ showAllComments[post._id] ? 'Hide comments' : 'Show all comments' }}
        </button>
        <br />
        <h2>New Comment</h2>
        <input
          class="comment"
          type="text"
          v-model="commentContent[index]"
          name="addComment"
          placeholder="Votre commentaire"
          minlength="1"
        />
        <button @click="addComment(post._id, commentContent[index], index)">Add comment</button>
        <input type="file" @change="addAttachement(post, $event)" accept=".jpg, .jpeg, .png" />

        <span class="conflicts" v-if="post._conflicts">Conflits détectés</span>
      </form>
    </article>
  </div>
  <form id="addPost" name="addPost" @submit.prevent="addPost(postTitle, postContent, postCategory)">
    <label for="sendId">Add New Post</label><br />
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
