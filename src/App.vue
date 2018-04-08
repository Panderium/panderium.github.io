<template>
  <div>
    <large-title></large-title>
    <presentation></presentation>
  </div>

</template>

<script>
import Title from './components/Title.vue'
import Content from './components/Content.vue';

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
