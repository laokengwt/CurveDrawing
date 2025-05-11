<script setup>
import HelloWorld from './components/HelloWorld.vue'
import TheWelcome from './components/TheWelcome.vue'
</script>

<template>
  <div id="app">
    <CurveDrawer />
  </div>
</template>

<script>
import CurveDrawer from './components/CurveDrawer.vue';

export default {
  name: 'App',
  components: {
    CurveDrawer
  }
};
</script>

<style>
body {
  margin: 0;
  padding: 20px;
  background-color: #f5f5f5;
}
</style>