<template>
<div>
  <large-title></large-title>
  <div class="container">
    <presentation></presentation>
  </div>
</div>

</template>

<script>
import Title from './Title.vue'
import Content from './Content.vue';

export default {

  components: {
    'presentation': Content,
    'large-title': Title
  },

  data () {
    return {
      msg: "dsfosd"
    }
  }
}
</script>
