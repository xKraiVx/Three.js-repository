<template>
  <main>
    <navigation />
    <router-view v-slot="{ Component }">
      <transition name="fade" mode="out-in">
        <component :is="Component" />
      </transition>
    </router-view>
    <stats/>
  </main>
</template>
<script>
import Navigation from "./components/Navigation.vue";
import Stats from "./components/Stats.vue";

export default {
  components: {
    Navigation,
    Stats
  },
};
</script>

<style lang="sass">
@import "./assets/scss/style.scss";
</style>
