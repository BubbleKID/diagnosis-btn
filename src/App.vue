<template>
  <div id="app">
    <DialogCustom/>
  </div>
</template>

<script>
import DialogCustom from './components/DialogCustom.vue';

export default {
  name: 'App',
  components: {
    DialogCustom,
  },
};
</script>

<style lang="scss">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
