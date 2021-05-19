<template>
  <b-modal
    id="workflow"
    title="Workflow configuration"
    size="lg"
  >
    <template #modal-title>
      Workflow Configuration

      <c-permissions-button
        v-if="workflow.canGrant"
        :title="workflow.meta.name || workflow.handle"
        :target="workflow.meta.name || workflow.handle"
        :resource="`automation:workflow:${workflow.workflowID}`"
        link
        class="ml-2"
      />
    </template>

    <div
      class="d-flex mb-3"
    >
      <import
        :disabled="importProcessing"
        @import="$emit('import', $event)"
      />

      <export
        v-if="workflow.workflowID && workflow.workflowID !== '0'"
        :workflows="[workflow.workflowID]"
        :file-name="workflow.handle"
        class="ml-1"
      />
    </div>

    <b-form-group
      label="Label"
    >
      <b-form-input
        v-model="workflow.meta.name"
        @change="$root.$emit('change-detected')"
      />
    </b-form-group>

    <b-form-group
      label="Handle"
    >
      <b-form-input
        v-model="workflow.handle"
        :state="handleState"
        @change="$root.$emit('change-detected')"
      />
    </b-form-group>

    <b-form-group
      label="Description"
    >
      <b-form-textarea
        v-model="workflow.meta.description"
        @change="$root.$emit('change-detected')"
      />
    </b-form-group>

    <b-form-group
      label="Run as"
      description="When not explicitly set, workflow will be executed by user that invoked it"
    >
      <vue-select
        :options="user.options"
        :get-option-label="getOptionLabel"
        :get-option-key="getOptionKey"
        :value="user.value"
        @search="search"
        @input="updateRunAs"
      />
    </b-form-group>

    <b-form-group>
      <b-form-checkbox
        v-model="workflow.enabled"
        @change="$root.$emit('change-detected')"
      >
        Enabled
      </b-form-checkbox>
    </b-form-group>

    <template #modal-footer>
      <div
        class="d-flex w-100"
      >
        <c-input-confirm
          v-if="workflow.canDeleteWorkflow"
          size="md"
          size-confirm="md"
          :borderless="false"
          @confirmed="$emit('delete')"
        >
          Delete
        </c-input-confirm>

        <b-button
          variant="primary"
          class="ml-auto"
          @click="$emit('save')"
        >
          Save
        </b-button>
      </div>
    </template>
  </b-modal>
</template>

<script>
import Import from '../Import'
import Export from '../Export'
import { debounce } from 'lodash'
import { VueSelect } from 'vue-select'

export default {
  components: {
    Import,
    Export,
    VueSelect,
  },

  props: {
    workflow: {
      type: Object,
      default: () => {},
    },
  },

  data () {
    return {
      importProcessing: false,

      user: {
        options: [],
        value: undefined,

        filter: {
          query: null,
          limit: 10,
        },
      },
    }
  },

  computed: {
    handleState () {
      const { handle } = this.workflow
      if (!handle || handle.length === 0 || handle.length > 64) {
        return false
      }

      return /^[A-Za-z][0-9A-Za-z_\-.]*[A-Za-z0-9]$/.test(handle)
    },
  },

  created () {
    if (this.workflow.runAs) {
      this.fetchUsers()
      this.getUserByID()
    }
  },

  methods: {
    search: debounce(function (query) {
      if (query !== this.user.filter.query) {
        this.user.filter.query = query
        this.user.filter.page = 1
      }

      if (query) {
        this.fetchUsers()
      }
    }, 300),

    fetchUsers () {
      this.$SystemAPI.userList(this.user.filter)
        .then(({ set }) => {
          this.user.options = set.map(m => Object.freeze(m))
        })
    },

    async getUserByID () {
      if (this.workflow.runAs !== '0') {
        this.$SystemAPI.userRead({ userID: this.workflow.runAs })
          .then(user => {
            this.user.value = user
          }).catch(() => {
            return {}
          })
      }
    },

    updateRunAs (user) {
      if (user && user.userID) {
        this.user.value = user
        this.workflow.runAs = user.userID
      } else {
        this.user.value = null
        this.workflow.runAs = '0'
      }
      this.$root.$emit('change-detected')
    },

    getOptionKey ({ userID }) {
      return userID
    },

    getOptionLabel ({ userID, email, name, username }) {
      return name || username || email || `<@${userID}>`
    },
  },
}
</script>
