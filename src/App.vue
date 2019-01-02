<template>
  <div id="app">
    <div>
    <h1>App.vue</h1>
    </div>
    <VoxelPainter title="Painter"/>
  </div>
</template>

<script>
import VoxelPainter from './components/VoxelPainter.vue'


export default {
  name: 'app',
  components: {
    VoxelPainter
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
