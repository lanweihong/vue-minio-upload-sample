<template>
  <div id="app">
    <minio-upload />
  </div>
</template>

<script>
import MinioUpload from './components/MinioUpload.vue'

export default {
  name: 'App',
  components: {
    MinioUpload
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
