<template>
  <div>
    <router-view v-slot="{ Component }">
      <transition name="fade" mode="out-in">
        <component :is="Component" />
      </transition>
    </router-view>
  </div>
</template>
<script>

export default {
  components: {
  },
};
</script>

<style lang="sass">
@import "./assets/scss/style.scss";
</style>
