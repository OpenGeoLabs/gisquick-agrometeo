<template>
  <div v-if="visible" class="legend-panel card">
    <div class="header dark f-row-ac">
      <translate class="mx-2">Legend</translate>
    </div>
    <map-legend :visible="visible"/>
  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import MapLegend from './content-panel/Legend.vue'

export default {
  name: 'LegendPanel',
  components: { MapLegend },
  computed: {
    ...mapGetters(['visibleLayers']),
    visible () {
      return this.visibleLayers.some(l => !l.hidden)
    }
  }
}
</script>

<style lang="scss" scoped>
.legend-panel {
  min-width: 200px;
  grid-column: 2 / 3;
  grid-row: 1 / 2;
  align-self: end;
  justify-self: start;
  z-index: 1;
  // top: 120px;
  left: 2px;
  bottom: 38px;
  background-color: #fff;
  overflow: hidden;
  .header {
    background-color: var(--color-dark);
    padding-block: 2px;
    font-size: 15px;
  }
}
</style>