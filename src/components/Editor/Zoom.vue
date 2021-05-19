<template>
  <div
    class="d-flex align-items-baseline p-2 bg-white border border-secondary"
  >
    {{ getZoomPercent }}
    <b-button
      variant="link"
      class="ml-4 p-0"
      @click="$emit('zoom-out')"
    >
      <font-awesome-icon
        :icon="['fas', 'search-minus']"
      />
    </b-button>
    <b-button
      variant="link"
      class="ml-1 p-0"
      @click="$emit('zoom-in')"
    >
      <font-awesome-icon
        :icon="['fas', 'search-plus']"
        class="pointer"
        @click="$emit('zoom-in')"
      />
    </b-button>
    <b-button
      variant="link"
      class="ml-2 p-0 text-decoration-none"
      @click="$emit('reset')"
    >
      Reset
    </b-button>
  </div>
</template>

<script>
export default {
  props: {
    zoomLevel: {
      type: Number,
      required: true,
    },
  },

  computed: {
    getZoomPercent () {
      return `${Math.floor(this.zoomLevel * 100).toFixed(0)}%`
    },
  },
}
</script>

<style>

</style>
