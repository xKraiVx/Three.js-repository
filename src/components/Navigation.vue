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
  computed: {
    currentRoute: function () {
      return this.$router.currentRoute.value.path;
    },
    nextEnable: function () {
      let currentRouteIdx = this.routes.indexOf(
        this.routes.filter((element) => element.path === this.currentRoute)[0]
      );
      if (currentRouteIdx === this.routes.length - 1) {
        return false;
      }
      return true;
    },
    prevEnable: function () {
      let currentRouteIdx = this.routes.indexOf(
        this.routes.filter((element) => element.path === this.currentRoute)[0]
      );
      if (currentRouteIdx === 0) {
        return false;
      }
      return true;
    },
  },
  mounted() {
    window.addEventListener("keyup", (e) => {
      if (e.key === "ArrowUp") {
        this.prev();
      }
      if (e.key === "ArrowDown") {
        this.next();
      }
    });
  },
  methods: {
    next() {
      if (!this.nextEnable) {
        return;
      }
      let currentRouteIdx = this.routes.indexOf(
          this.routes.filter((element) => element.path === this.currentRoute)[0]
        ),
        nextRoute = this.routes[currentRouteIdx + 1];
      this.$router.push(nextRoute);
    },
    prev() {
      if (!this.prevEnable) {
        return;
      }
      let currentRouteIdx = this.routes.indexOf(
        this.routes.filter((element) => element.path === this.currentRoute)[0]
      );
      let nextRoute = this.routes[currentRouteIdx - 1];
      this.$router.push(nextRoute);
    },
  },
};
</script>
<style scoped lang="scss">
@import "../assets/scss/default/colors.scss";
@import "../assets/scss/components/navigation.scss";
@import "../assets/scss/components/arrow.scss";
</style>