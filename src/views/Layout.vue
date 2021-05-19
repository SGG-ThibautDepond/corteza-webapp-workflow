<template>
  <div class="d-flex flex-column w-100 vh-100">
    <header>
      <c-topbar
        :sidebar-pinned="pinned"
      >
        <template #title>
          <portal-target name="topbar-title" />
        </template>
      </c-topbar>
    </header>

    <aside>
      <c-sidebar
        :expanded.sync="expanded"
        :pinned.sync="pinned"
        :expand-on-hover="expandOnHover"
      >
        <template #header-collapsed>
          <portal-target name="sidebar-header-collapsed" />
        </template>

        <template #header-expanded>
          <portal-target name="sidebar-header-expanded" />
        </template>

        <template #body-collapsed>
          <portal-target name="sidebar-body-collapsed" />
        </template>

        <template #body-expanded>
          <portal-target name="sidebar-body-expanded" />
        </template>

        <template #footer-collapsed>
          <portal-target name="sidebar-footer-collapsed" />
        </template>

        <template #footer-expanded>
          <portal-target name="sidebar-footer-expanded" />
        </template>
      </c-sidebar>
    </aside>

    <main class="d-inline-flex w-100 h-100">
      <!--
        Content spacer
        Large and xl screens should push in content when the nav is expanded
      -->
      <template>
        <div
          class="spacer"
          :class="{
            'expanded': expanded && pinned,
          }"
        />
      </template>

      <router-view />
    </main>

    <c-permissions-modal />
  </div>
</template>

<script>
import { components } from '@cortezaproject/corteza-vue'
const { CPermissionsModal, CTopbar, CSidebar } = components

export default {
  components: {
    CTopbar,
    CSidebar,
    CPermissionsModal,
  },

  data () {
    return {
      // Sidebar and Topbar
      expanded: false,
      pinned: false,
      expandOnHover: true,
    }
  },

  created () {
    this.$root.$on('alert', this.displayToast)
    this.$root.$on('set-expand-onhover', this.setExpandOnHover)
  },

  methods: {
    setExpandOnHover (value) {
      this.expandOnHover = value
    },

    displayToast ({ title, message, variant, countdown }) {
      this.$bvToast.toast(message, {
        title,
        variant,
        solid: true,
        autoHideDelay: countdown,
        toaster: 'b-toaster-bottom-right',
      })
    },
  },
}
</script>

<style lang="scss" scoped>
main {
  padding-top: $topbar-height;
}

.spacer {
  min-width: 77px;
  transition: width 0.1s ease-in-out;

  &.expanded {
    min-width: $sidebar-width;
  }
}
</style>
