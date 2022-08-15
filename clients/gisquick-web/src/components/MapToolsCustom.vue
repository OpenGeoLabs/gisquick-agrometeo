<script>
import Vue from 'vue'

import MapTools from './MapTools.vue'
import Identification from './Identification'

export default {
  extends: MapTools,
  computed: {
    hiddenIdentificationTool () {
      return {
        name: 'hidden-identification',
        props: Vue.observable({
          identificationLayer: '',
          displayMode: 'info-panel'
        }),
        component: Identification
      }
    },
    items () {
      return [
        this.hiddenIdentificationTool,
        this.measureTool,
        this.printTool,
        this.attributeTableTool
      ]
    },
  },
  watch: {
    activeTool: {
      immediate: true,
      handler (activeTool) {
        if (!activeTool) {
          this.$store.commit('activeTool', 'identification')
        }
      }
    }
  }
}
</script>
