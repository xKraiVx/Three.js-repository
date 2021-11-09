<template>
  <main class="home">
    <h1>Fluppy bird on THREE.js and CANNON.js</h1>
    <router-link class="btn" :to="routes[1].path">Start game</router-link>
  </main>
</template>
<script>
export default {
  data: () => ({
    routes: [],
  }),
  created() {
    this.routes = this.$router.options.routes;
  },
};
</script>
<style lang="sass" scoped>
@import "../assets/scss/default/colors.scss"
@import "../assets/scss/pages/home.scss"
</style>
