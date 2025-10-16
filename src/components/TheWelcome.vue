<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

const storage = ref()
// const postData = ref([])

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
})

const initDatabase = () => {
  console.log('=> Connexion à la base de données.')
  const db = new PouchDB('http://admin_loann:2B$8a#oq7z89E9#g@localhost:5984/infra_53_0850')
  if (db) {
    console.log('Connected to collection : ' + db?.name)
    storage.value = db
  } else {
    console.warn('Something went wrong')
  }
}
</script>

<template>
  <h1>Cookie Clicker!</h1>
</template>
