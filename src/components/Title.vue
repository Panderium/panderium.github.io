<template lang="html">
  <section class="hero is-primary is-medium">
    <div class="hero-head">
      <navbar></navbar>
    </div>
    <div class="hero-body">
      <profile></profile>
    </div>
  </section>
</template>

<script>
import Navbar from './Navbar.vue'
import Profile from './Profile.vue'

export default {
  components: {
    'navbar': Navbar,
    'profile': Profile,
  }

}
</script>

<style lang="css">
</style>
