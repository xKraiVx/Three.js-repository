<template>
  <div class="popup">
    <div class="popup__content">
      <h1 v-if="content.h1" v-html="content.h1" />
      <p v-if="content.p" v-html="content.p" />
      <strong
        v-if="content.strong"
        v-html="isMobile ? content.strongMobile : content.strong"
      />
    </div>
  </div>
</template>

<script>
export default {
  props: {
    content: Object,
    isMobile: Boolean,
  },
};
</script>

<style lang="scss" scoped>
@import "../assets/scss/default/colors.scss";
@import "../assets/scss/components/popup.scss";
</style>
