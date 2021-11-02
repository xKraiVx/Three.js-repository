<template>
  <nav class="navigation" id="nav">
    <div class="navigation__wrapper">
      <ul class="navigation__list">
        <li
          class="navigation__list-item"
          v-for="(route, idx) in routes"
          :key="route.path"
        >
          <router-link :to="route.path">{{ content[idx] }} </router-link>
        </li>
      </ul>
    </div>
  </nav>
</template>

<script>
import json from "../data/data.json";

export default {
  data: () => ({
    routes: [],
    content: json.navigation,
  }),
  created() {
    this.routes = this.$router.options.routes;
  },
};
</script>
<style scoped lang="scss">
@import "../assets/scss/default/colors.scss";
@import "../assets/scss/components/navigation.scss";
@import "../assets/scss/components/arrow.scss";
</style>