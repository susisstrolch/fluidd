<template>
  <v-snackbar
    v-model="open"
    :color="type"
    :timeout="timeout"
    elevation="24"
    multi-line
    top
    centered
  >
    <span v-html="text" />

    <template #action="{ attrs }">
      <app-btn
        dark
        v-bind="attrs"
        @click="open = false"
      >
        {{ $t('app.general.btn.close') }}
      </app-btn>
    </template>
  </v-snackbar>
</template>

<script lang="ts">
import Vue from 'vue'
import { Component, Prop } from 'vue-property-decorator'

@Component({})
export default class FlashMessage extends Vue {
  @Prop({ type: Boolean })
  value!: boolean;

  @Prop({ type: String, default: 'dark' })
  type!: string;

  @Prop({ type: String, default: 'Saved!' })
  text!: string;

  @Prop({ type: Number, default: 1500 })
  timeout!: number;

  get open () {
    return this.$props.value
  }

  set open (value: boolean) {
    this.$emit('input', value)
  }
}
</script>

<style lang="scss" scoped>
  ::v-deep .v-snack__wrapper .v-snack__content {
    overflow: hidden;
    overflow-wrap: break-word;
  }
</style>
