<script>
export default {
  props: {
    item: {
      type: Object,
      default: () => {},
    },

    edges: {
      type: Object,
      default: () => {},
    },

    outEdges: {
      type: Number,
      default: 0,
    },
  },
}
</script>
