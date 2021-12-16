<script setup>
  import { ref, onBeforeMount } from 'vue'
  import { get, set, del, getMany } from 'idb-keyval'

  let url = ref('')
  let disabled = ref(false)
  let keys = new Set()
  let mp3BlobMap = new Map()
  let mp3BlobArray = ref([])
  let freeStorage = ref(undefined)

  async function estimate() {
    const estimate = await navigator.storage.estimate()
    freeStorage.value = ((estimate.quota - estimate.usage) / 1_000_000).toFixed(0) + 'MB'
  }

  async function getData() {
    const keysTmp = await get('keys')
    if (keysTmp) {
      keys = keysTmp
      const keysArray = Array.from(keys)
      const valueArray = await getMany(keysArray)
      for (let i = 0; i < keysArray.length; i++) {
        mp3BlobMap.set(keysArray[i], URL.createObjectURL(valueArray[i]))
        convert()
      }
    }
  }

  onBeforeMount(async () => {
    await estimate()
    await getData()
  })

  async function add() {
    disabled.value = true
    let response = await fetch(url.value)
    let blob = await response.blob()
    const newKey = badId()
    keys.add(newKey)
    await set(newKey, blob)
    await set('keys', keys)
    mp3BlobMap.set(newKey, URL.createObjectURL(blob))
    convert()
    estimate()
    disabled.value = false
    url.value = ''
  }

  async function delMp3(key) {
    await del(key)
    mp3BlobMap.delete(key)
    convert()
    keys.delete(key)
    await set('keys', keys)
  }

  function badId() {
    return String((new Date()).getTime())
  }

  function convert() {
    mp3BlobArray.value = []
    for (let entry of mp3BlobMap) {
      mp3BlobArray.value.push({key: entry[0], value: entry[1]})
    }
  }
</script>

<template>
  <div class="mx-auto max-w-md">
    <h1 class="my-7 text-center text-xl">{{ freeStorage }} free storage space</h1>
    <div class="flex flex-row justify-center items-center">
      <input v-model="url" type="text" class="border-2 py-2 px-4 mx-4 rounded"
             placeholder="Insert URL to MP3 file with the allowed CORS"
             :disabled="disabled">
      <button class="bg-sky-500 text-white font-bold py-2 px-4 rounded disabled:bg-gray-200 disabled:text-gray-400 disabled:cursor-not-allowed"
              @click="add" :disabled="disabled">Add mp3</button>
    </div>
    <div v-for="entry in mp3BlobArray" class="flex flex-row justify-center items-center">
      <audio controls :src="entry.value" class="m-3"></audio>
      <button class="text-red-500 font-bold py-2 px-4 rounded cursor-pointer border-2 border-red-500"
              @click="delMp3(entry.key)">X</button>
    </div>
  </div>
</template>

<style>
</style>
