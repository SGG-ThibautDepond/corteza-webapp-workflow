<template>
  <b-modal
    id="issues"
    v-model="showIssues"
    title="Step issues"
    hide-footer
  >
    <div
      v-for="(issue, index) in issues"
      :key="index"
    >
      <p>
        <code>{{ issue[0].toUpperCase() + issue.slice(1).toLowerCase() }}</code>
      </p>
    </div>
  </b-modal>
</template>

<script>
export default {
  props: {
    show: {
      type: Boolean,
      required: true,
    },

    issues: {
      type: Array,
      default: () => [],
    },
  },

  computed: {
    showIssues: {
      get () {
        return this.show
      },

      set (show) {
        this.$emit('update:show', show)
      },
    },
  },
}
</script>

<style>

</style>
