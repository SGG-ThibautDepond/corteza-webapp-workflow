<template>
  <!--
    no-enforce-focus flag doesn't set focus to sidebar when it is opened.
    Bad for Accesability, since keyboard only users can't use sidebar.
  -->
  <b-sidebar
    v-model="show"
    shadow
    right
    lazy
    no-enforce-focus
    width="500px"
    header-class="bg-white border-bottom border-light p-2"
    body-class="bg-white"
    footer-class="bg-white border-top border-light p-1"
  >
    <template
      #header
    >
      <div
        v-if="item"
        class="d-flex align-items-center w-100 h5 mb-0 p-2"
      >
        <b-img
          :src="getItemIcon"
        />
        <h4
          class="text-primary font-weight-bold ml-2 mb-0"
        >
          <b>{{ getItemType }}</b>
        </h4>

        <div
          class="ml-auto"
        >
          ID: <var>{{ getItemID }}</var>
        </div>
      </div>
    </template>

    <slot />

    <template
      #footer
    >
      <div
        class="d-flex m-2"
      >
        <c-input-confirm
          size="md"
          size-confirm="md"
          variant="danger"
          :borderless="false"
          @confirmed="$emit('delete')"
        >
          Delete
        </c-input-confirm>
      </div>
    </template>
  </b-sidebar>
</template>

<script>
import { getStyleFromKind } from '../../lib/editor/style'

export default {
  props: {
    show: {
      type: Boolean,
      required: true,
    },

    item: {
      type: Object,
      required: true,
    },
  },

  computed: {
    getItemID () {
      return this.item.node ? this.item.node.id : undefined
    },

    getItemIcon () {
      if (this.item && this.item.config) {
        return getStyleFromKind(this.item.config).icon
      }

      return undefined
    },

    getItemType () {
      if (this.item && this.item.config) {
        const itemType = this.item.node.edge ? 'edge' : this.item.config.kind

        if (itemType) {
          if (itemType === 'edge') {
            return 'Connector'
          }
          return itemType.charAt(0).toUpperCase() + itemType.slice(1)
        }

        return itemType
      }
      return undefined
    },
  },
}
</script>

<style>

</style>
