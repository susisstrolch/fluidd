<template>
  <collapsable-card
    :title="$tc('app.general.title.gcode_preview')"
    icon="$cubeScan"
    layout-path="dashboard.gcode-preview-card"
    draggable
  >
    <v-card-text v-if="file">
      {{ file.name }}
    </v-card-text>

    <v-divider v-if="file" />

    <v-card-text>
      <GcodePreviewParserProgressDialog
        :value="showParserProgressDialog"
        :progress="parserProgress"
        :file="file"
        @cancel="abortParser"
      />

      <v-row>
        <v-col
          cols="12"
          lg="9"
          md="7"
        >
          <v-row>
            <v-col>
              <app-slider
                :label="$t('app.gcode.label.layer')"
                :value="currentLayer + 1"
                :min="(!fileLoaded) ? 0 : 1"
                :max="layerCount"
                :disabled="!fileLoaded"
                :locked="isMobile"
                input-md
                @input="setCurrentLayer($event - 1)"
              />
            </v-col>
          </v-row>
          <v-row>
            <v-col>
              <app-slider
                :label="$t('app.general.label.progress')"
                :value="moveProgress - currentLayerMoveRange.min"
                :min="0"
                :max="currentLayerMoveRange.max - currentLayerMoveRange.min"
                :disabled="!fileLoaded"
                value-suffix="moves"
                :locked="isMobile"
                input-md
                @input="setMoveProgress($event + currentLayerMoveRange.min)"
              />
            </v-col>
          </v-row>
          <v-row>
            <v-col>
              <gcode-preview
                width="100%"
                :layer="currentLayer"
                :progress="moveProgress"
                :disabled="!fileLoaded"
              />
            </v-col>
          </v-row>
        </v-col>
        <v-col
          cols="12"
          lg="3"
          md="5"
        >
          <v-card
            outlined
            class="px-2 py-1 text-center stat-square"
          >
            <div class="">
              {{ $t('app.gcode.label.layers') }}
            </div>
            <div class="focus--text">
              {{ layerCount }}
            </div>
            <div class="">
              {{ $t('app.gcode.label.current_layer_height') }}
            </div>
            <div class="focus--text">
              {{ currentLayerHeight }}
            </div>
          </v-card>
          <GcodePreviewControls :disabled="!fileLoaded" />
        </v-col>
      </v-row>
    </v-card-text>
  </collapsable-card>
</template>

<script lang="ts">
import { Component, Mixins, Prop, Watch } from 'vue-property-decorator'
import StateMixin from '@/mixins/state'
import FilesMixin from '@/mixins/files'
import GcodePreview from './GcodePreview.vue'
import GcodePreviewControls from '@/components/widgets/gcode-preview/GcodePreviewControls.vue'
import { AppFile } from '@/store/files/types'
import GcodePreviewParserProgressDialog from '@/components/widgets/gcode-preview/GcodePreviewParserProgressDialog.vue'
import { MinMax } from '@/store/gcodePreview/types'

@Component({
  components: {
    GcodePreviewParserProgressDialog,
    GcodePreview,
    GcodePreviewControls
  }
})
export default class GcodePreviewCard extends Mixins(StateMixin, FilesMixin) {
  @Prop({
    type: Boolean,
    default: true
  })
  enabled!: boolean

  currentLayer = 0
  moveProgress = 0

  get visibleLayer () {
    return this.currentLayer + 1
  }

  @Watch('layerCount')
  onLayerCountChanged () {
    this.currentLayer = 0
  }

  @Watch('followProgress')
  onFollowProgressChanged () {
    if (this.followProgress) {
      this.currentLayer = this.fileProgressLayerNr
      this.syncMoveProgress()
    }
  }

  @Watch('currentLayer')
  onCurrentLayerChanged () {
    if (this.followProgress && this.currentLayer !== this.fileProgressLayerNr) {
      this.followProgress = false
    }

    if (!this.followProgress) {
      this.moveProgress = this.currentLayerMoveRange.max
    }
  }

  @Watch('filePosition')
  onFilePositionChanged () {
    if (this.followProgress) {
      this.syncMoveProgress()

      const moves = this.$store.getters['gcodePreview/getMoves']
      const {
        min,
        max
      } = this.currentLayerMoveRange

      if (this.filePosition < moves[min].filePosition || this.filePosition > moves[max].filePosition) {
        this.currentLayer = this.fileProgressLayerNr
      }
    }
  }

  @Watch('moveProgress')
  onMoveProgressChanged () {
    if (this.followProgress) {
      const fileMovePosition = this.$store.getters['gcodePreview/getMoveIndexByFilePosition'](this.filePosition)

      // In some (yet unclear) cases, fileMovePosition can get out of sync with
      // the component's notion of moveProgress.  This seems to happen during
      // layer changes, but not every time.  Possibly some gcode command is getting
      // misinterpreted.
      // This "fix" simply forces a re-sync of progress if they get out of sync
      if (fileMovePosition !== this.moveProgress) {
        this.syncMoveProgress()
      }
    }
  }

  get file (): AppFile | undefined {
    return this.$store.getters['gcodePreview/getFile']
  }

  get fileLoaded (): boolean {
    return this.$store.getters['gcodePreview/getMoves'].length > 0
  }

  get isMobile (): boolean {
    return this.$vuetify.breakpoint.mobile
  }

  get parserProgress (): number {
    return this.$store.getters['gcodePreview/getParserProgress']
  }

  get showParserProgressDialog (): boolean {
    return this.file !== undefined && this.parserProgress !== this.file.size
  }

  get filePosition (): number {
    return this.$store.state.printer.printer.virtual_sdcard.file_position
  }

  get fileProgressLayerNr (): number {
    return this.$store.getters['gcodePreview/getLayerNrByFilePosition'](this.filePosition)
  }

  get layerCount (): number {
    return this.$store.getters['gcodePreview/getLayers'].length
  }

  get currentLayerHeight (): number {
    return this.$store.getters['gcodePreview/getLayers'][this.currentLayer - 1]?.z ?? 0
  }

  get followProgress (): boolean {
    return this.$store.getters['gcodePreview/getViewerOption']('followProgress')
  }

  set followProgress (value) {
    this.$store.commit('gcodePreview/setViewerState', { followProgress: value })
  }

  get currentLayerMoveRange (): MinMax {
    const moves = this.$store.getters['gcodePreview/getMoves']

    if (moves.length === 0) {
      return {
        min: 0,
        max: 0
      }
    }

    const layers = this.$store.getters['gcodePreview/getLayers']

    return {
      min: layers[this.currentLayer].move,
      max: layers[this.currentLayer + 1]?.move ?? moves.length - 1
    }
  }

  setCurrentLayer (value: number) {
    if (value > 0) this.currentLayer = value
  }

  setMoveProgress (value: number) {
    if (value >= 0) this.moveProgress = value
  }

  syncMoveProgress () {
    this.moveProgress = this.$store.getters['gcodePreview/getMoveIndexByFilePosition'](this.filePosition)
  }

  abortParser () {
    this.$store.dispatch('gcodePreview/terminateParserWorker')
  }
}
</script>
