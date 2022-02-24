<template>
  <div>
    <MapGraphic width="100%" height="500px"/>
  </div>
</template>

<script>
// @ is an alias to /src
import MapGraphic from '@/components/MapGraphic.vue'

export default {
  name: 'MapView',
  components: {
    MapGraphic
  }
}
</script>
