
<template>
  <div v-if="feature" class="f-col">
    <slot
      name="form"
      :fields="fields"
      :layer="layer"
    >
      <generic-edit-form
        ref="editForm"
        :layer="layer"
        :initial="originalFields"
        :fields="fields"
        :project="project"
        :status.sync="formStatus"
      />
      <!-- <v-radio-btn val="loading" v-model="status" label="Loading"/>
      <v-radio-btn val="success" v-model="status" label="Success"/>
      <v-radio-btn val="error" v-model="status" label="Error"/>
      <v-radio-btn val="" v-model="status" label="None"/> -->
    </slot>
    <portal to="infopanel-tool">
      <div class="toolbar f-row-ac">
        <v-btn
          class="icon flat"
          :color="editGeometry ? 'primary' : ''"
          @click="editGeometry = !editGeometry"
        >
          <v-tooltip slot="tooltip">
            <translate>Edit geometry</translate>
          </v-tooltip>
          <v-icon name="edit-geometry"/>
        </v-btn>
        <geometry-editor
          v-if="editGeometry"
          ref="geometryEditor"
          :feature="editGeometryFeature"
          :geometry-type="geomType"
        />
        <div class="v-separator"/>
        <!-- <v-btn @click="deleteFeature" icon>
          <v-icon color="red darken-3">delete_forever</v-icon>
        </v-btn> -->

        <v-btn
          aria-label="Delete object"
          class="icon"
          :disabled="!permissions.delete || status === 'loading'"
          @click="showConfirmDelete = true"
        >
          <v-icon color="red" name="delete_forever"/>
          <v-tooltip slot="tooltip">
            <translate>Delete object</translate>
          </v-tooltip>
        </v-btn>
        <v-btn
          class="icon"
          :disabled="!permissions.update || !isModified || !!status"
          @click="restore"
        >
          <v-tooltip slot="tooltip">
            <translate>Discard changes</translate>
          </v-tooltip>
          <v-icon color="orange" name="restore"/>
        </v-btn>
        <v-btn
          class="icon"
          :disabled="!isModified || !!status || formStatus === 'error'"
          @click="save"
        >
          <v-tooltip slot="tooltip">
            <translate>Save changes</translate>
          </v-tooltip>
          <v-icon color="green" name="save"/>
        </v-btn>
      </div>
    </portal>

    <transition name="fade">
      <div v-if="status" class="overlay notification f-row">
        <div
          class="content shadow-2 f-row-ac p-2"
          :class="status"
        >
          <progress-action class="mr-2" :status="status"/>
          <translate v-if="status === 'loading'" key="pending">Updating data</translate>
          <translate v-else-if="status === 'success'" key="success">Data updated</translate>
          <translate v-else key="error">Error</translate>
        </div>
      </div>
    </transition>
    <transition name="fade">
      <div v-if="showConfirmDelete" class="overlay delete-dialog f-col">
        <!-- <div class="content shadow-2"> -->
          <div class="header px-4">
            <translate class="title">Delete current object?</translate>
          </div>
          <div class="f-row-ac">
            <v-btn class="small round f-grow" color="#777" @click="showConfirmDelete = false">
              <translate>No</translate>
            </v-btn>
            <v-btn class="small round f-grow" color="red" @click="deleteFeature">
              <translate>Yes</translate>
            </v-btn>
          </div>
        <!-- </div> -->
      </div>
    </transition>
  </div>
</template>

<script>
import omit from 'lodash/omit'
import isEqual from 'lodash/isEqual'
import Style from 'ol/style/Style'
import Feature from 'ol/Feature'

import { wfsTransaction } from '@/map/wfs'
import { queuedUpdater } from '@/utils'
import GeometryEditor from './GeometryEditor.vue'
import GenericEditForm from './GenericEditForm.vue'
import ProgressAction from '@/components/ProgressAction.vue'

function getFeatureFields (feature) {
  return feature ? omit(feature.getProperties(), feature.getGeometryName()) : {}
}

function difference (obj1, obj2) {
  const diff = {}
  Object.keys(obj1).forEach(key => {
    if (!isEqual(obj1[key], obj2[key])) {
      diff[key] = obj1[key]
    }
  })
  return diff
}

const HiddenStyle = new Style()

export default {
  name: 'FeatureEditor',
  components: { GeometryEditor, GenericEditForm, ProgressAction },
  refs: ['geometryEditor'],
  props: {
    layer: Object,
    feature: Object,
    project: Object
  },
  data () {
    return {
      status: '',
      errorMsg: '',
      fields: null,
      originalFields: null,
      editGeometry: false,
      showConfirmDelete: false,
      formStatus: null
    }
  },
  computed: {
    geomType () {
      const geom = this.feature && this.feature.getGeometry()
      if (geom) {
        return geom.getType()
      }
      return this.layer.wkb_type || {
        POINT: 'MultiPoint',
        LINE: 'MultiLineString',
        POLYGON: 'MultiPolygon'
      }[this.layer.geom_type]
    },
    fieldsModified () {
      return !isEqual(this.fields, this.originalFields)
    },
    geomModified () {
      const editor = this.$refs.geometryEditor
      return editor && editor.geomModified
    },
    isModified () {
      return this.fieldsModified || this.geomModified
    },
    editGeometryFeature () {
      return this.editGeometry && this.feature
    },
    permissions () {
      return this.layer.permissions || {}
    }
  },
  watch: {
    feature: {
      immediate: true,
      handler (feature, old) {
        this.fields = getFeatureFields(feature)
        this.originalFields = getFeatureFields(feature)
      }
    }
  },
  created () {
    let origStyle
    this.$watch('editGeometryFeature', (feature, prevFeature) => {
      if (prevFeature && prevFeature.getStyle() === HiddenStyle) {
        prevFeature.setStyle(origStyle)
      }
      if (feature) {
        origStyle = feature.getStyle()
        feature.setStyle(HiddenStyle)
      }
    })
    this.$once('hook:beforeDestroy', () => {
      if (this.editGeometryFeature && this.editGeometryFeature.getStyle() === HiddenStyle) {
        this.editGeometryFeature.setStyle(origStyle)
      }
    })
    this.statusController = queuedUpdater(v => { this.status = v })
  },
  beforeDestroy () {
    this.restore()
  },
  methods: {
    restore () {
      this.fields = getFeatureFields(this.feature)
      this.editGeometry = false
    },
    async wfsTransaction (operations) {
      this.statusController.set('loading', 1000)
      try {
        const { updates = [], deletes = [] } = operations
        await wfsTransaction(this.project.ows_url, this.layer.name, operations)
        for (const f of updates) {
          await this.$refs.editForm.afterFeatureUpdated?.(f)
        }
        for (const f of deletes) {
          await this.$refs.editForm.afterFeatureDeleted?.(f)
        }
        await this.statusController.set('success', 1500)
        this.statusController.set(null, 100)
        updates.forEach(f => this.$emit('edit', f))
        deletes.forEach(f => this.$emit('delete', f))
      } catch (err) {
        this.errorMsg = err.message || 'Error'
        this.statusController.set('error', 3000)
        this.statusController.set(null, 100)
      }
    },
    async save () {
      const f = new Feature()
      const resolvedFields = {}
      for (const name in this.fields) {
        let value = this.fields[name]
        if (typeof value === 'function') {
          try {
            value = await value()
          } catch (err) {
            // this.statusController.set('error', 3000)
            // this.statusController.set(null, 100)
            return
          }
        }
        resolvedFields[name] = value
      }
      const changedFields = difference(resolvedFields, this.originalFields)
      f.setProperties(changedFields)
      if (this.geomModified) {
        let newGeom = this.$refs.geometryEditor.getGeometry()
        const mapProjection = this.$map.getView().getProjection().getCode()
        if (mapProjection !== this.layer.projection) {
          newGeom = newGeom.clone()
          newGeom.transform(mapProjection, this.layer.projection)
        }
        f.setGeometry(newGeom)
      }
      f.setId(this.feature.getId())
      await this.$refs.editForm.beforeFeatureUpdated?.(f)
      this.wfsTransaction({ updates: [f] })
    },
    async deleteFeature () {
      await this.$refs.editForm.beforeFeatureDeleted?.(this.feature)
      this.wfsTransaction({ deletes: [this.feature] })
      this.showConfirmDelete = false
    }
  }
}
</script>

<style lang="scss" scoped>
.toolbar {
  background-color: #e0e0e0;
  border-top: 1px solid #bbb;
  ::v-deep .btn.icon {
    margin: 3px 2px;
    width: 26px;
    height: 26px;
  }
}

.overlay {
  position: absolute;
  inset: 0;
  align-items: center;
  justify-content: center;
  z-index: 10;
}
.notification {
  background-color: rgba(0, 0, 0, 0.2);

  // pointer-events: none;
  svg {
    border: 1px solid currentColor;
    border-radius: 50%;
    // color: var(--icon-color);
  }
  .content {
    min-width: 150px;
    max-width: 250px;
    font-size: 14px;
    border-radius: 3px;
    margin: 6px;
    background-color: #444;
    color: #fff;
    transition: 0.4s ease;
    &.error {
      background-color: var(--color-red);
    }
  }
}
.delete-dialog {
  background-color: #f3f3f3;
  // ver. 2 (with content wrapper)
  // background-color: rgba(0,0,0, 0.3);
  // .content {
  //   background-color: #f3f3f3;
  //   padding: 6px 12px;
  //   border-radius: 2px;
  // }
  .title {
    font-weight: 500;
  }
  .btn.small {
    min-width: 70px;
  }
}
</style>
