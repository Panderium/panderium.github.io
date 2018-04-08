<template>
  <div class="container is-fluid">
    <div class="container">
    <nav class="navbar">
      
      <desktop-navbar></desktop-navbar>

    </nav>
  </div>
    <div class="is-divider"></div>
  </div>

</template>

<script>
import MobileNavbar from './MobileNavbar.vue'
import DesktopNavbar from './DesktopNavbar.vue';
export default {

  components: {
    MobileNavbar,
    DesktopNavbar
  },
}
</script>

<style lang="css">
.is-divider {
  margin-top: 0;
}
</style>
