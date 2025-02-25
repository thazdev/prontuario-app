<template>
  <v-app>
    <TextEditor />
  </v-app>
</template>

<script>
import TextEditor from './components/TextEditor.vue';

export default {
  components: {
    TextEditor
  }
}
</script>

<style>
html, body, #app, .v-application {
  background-color: #242424 !important; 
  font-family: 'Poppins', sans-serif;
  color: #333333;
  height: 100%;
  margin: 0;
  padding: 0;
}
</style>
