<template>
  <div class="home">
    <NewMap/>
  </div>
</template>

<script>
// @ is an alias to /src
import NewMap from '@/components/NewMap.vue'

export default {
  name: 'Home',
  components: {
    NewMap
  }
}
</script>
