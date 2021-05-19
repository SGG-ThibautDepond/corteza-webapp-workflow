<template>
  <div
    class="d-flex h-100 editor"
  >
    <workflow-editor
      v-if="!processing"
      :workflow-object="workflow"
      :workflow-triggers="triggers"
      :change-detected="changeDetected"
      :can-create="canCreate"
      @save="saveWorkflow"
      @delete="deleteWorkflow"
    />

    <portal to="sidebar-header-expanded">
      <vue-select
        key="workflowID"
        label="handle"
        class="workflow-selector sticky-top bg-white my-2"
        :clearable="false"
        :options="filteredWorkflows"
        :value="currentWorkflow"
        @option:selected="workflowSelected"
      >
        <template v-slot:option="wf">
          <span class="text-truncate">
            {{ wf.meta.name || wf.handle }}
          </span>
        </template>
        <template #list-footer>
          <router-link
            :to="{ name: 'workflow.create' }"
            class="d-block mt-3 ml-3 mb-1 font-weight-bold"
          >
            + Add new
          </router-link>
        </template>
      </vue-select>
    </portal>
  </div>
</template>

<script>
import WorkflowEditor from '../../components/WorkflowEditor'
import { VueSelect } from 'vue-select'
import { automation } from '@cortezaproject/corteza-js'

export default {
  name: 'Editor',

  components: {
    VueSelect,
    WorkflowEditor,
  },

  data () {
    return {
      canCreate: false,

      workflows: [],

      processing: true,
      workflow: {},
      triggers: [],

      changeDetected: false,
    }
  },

  computed: {
    workflowID () {
      return this.$route.params.workflowID || (this.workflow.workflowID !== '0' ? this.workflow.workflowID : undefined)
    },

    filteredWorkflows () {
      return this.workflows.filter(({ workflowID }) => workflowID !== this.workflowID)
    },

    currentWorkflow () {
      const { workflowID, handle } = this.workflow
      return { workflowID, handle }
    },

    userID () {
      if (this.$auth.user) {
        return this.$auth.user.userID
      }
      return undefined
    },
  },

  watch: {
    workflowID: {
      async handler () {
        this.processing = true

        await this.fetchTriggers()
        await this.fetchWorkflow()

        this.changeDetected = false

        this.processing = false
      },
    },
  },

  async mounted () {
    this.$root.$emit('set-expand-onhover', false)
    window.onbeforeunload = null

    this.$root.$on('change-detected', () => {
      if (!this.changeDetected) {
        window.onbeforeunload = () => {
          return true
        }
      }

      this.changeDetected = true
    })

    this.fetchPermissions()
    this.fetchWorkflowList()

    if (this.workflowID) {
      await this.fetchTriggers()
      await this.fetchWorkflow()
    } else {
      this.workflow = new automation.Workflow({
        ownedBy: this.userID,
        runAs: '0',
        enabled: true,
        handle: '',
        meta: {
          name: 'Unnamed Workflow',
        },
      })
    }

    this.processing = false
  },

  beforeRouteLeave (to, from, next) {
    if (this.changeDetected) {
      next(window.confirm('You have unsaved changes, are you sure you want to exit?'))
    } else {
      window.onbeforeunload = null
      next()
    }
  },

  beforeDestroy () {
    window.onbeforeunload = null
  },

  methods: {
    async fetchWorkflow () {
      this.workflow = {}
      return this.$AutomationAPI.workflowRead({ workflowID: this.workflowID })
        .then(wf => {
          this.workflow = wf
        })
        .catch(this.defaultErrorHandler('Failed to fetch workflow'))
    },

    async fetchWorkflowList () {
      return this.$AutomationAPI.workflowList({ disabled: 1, sort: 'handle' })
        .then(({ set = [] }) => {
          this.workflows = set
        })
        .catch(this.defaultErrorHandler('Failed to fetch workflows'))
    },

    async fetchTriggers (workflowID = this.workflowID) {
      this.triggers = []
      return this.$AutomationAPI.triggerList({ workflowID, disabled: 1 })
        .then(({ set = [] }) => {
          this.triggers = set
        })
        .catch(this.defaultErrorHandler('Failed to fetch triggers'))
    },

    fetchPermissions () {
      this.$AutomationAPI.permissionsEffective()
        .then(rules => {
          this.canCreate = rules.find(({ resource, operation }) => resource === 'automation' && operation === 'workflow.create').allow
        })
        .catch(this.defaultErrorHandler('Failed to fetch automation permissions'))
    },

    workflowSelected ({ workflowID }) {
      if (this.changeDetected && !window.confirm('You have unsaved changes, are you sure you want to switch workflows?')) {

      } else {
        this.$router.push({ name: 'workflow.edit', params: { workflowID } })
      }
    },

    async saveWorkflow (wf) {
      try {
        const isNew = wf.workflowID === '0'

        const { steps = [], paths = [], triggers = [] } = wf

        this.workflow.steps = steps
        this.workflow.paths = paths

        if (isNew) {
          // Create workflow
          if (!this.canCreate) {
            throw new Error('Not allowed to create workflow')
          }

          wf = await this.$AutomationAPI.workflowCreate(wf)
        } else {
          if (!this.workflow.canUpdateWorkflow) {
            throw new Error('Not allowed to update workflow')
          }

          wf = await this.$AutomationAPI.workflowUpdate(wf)
        }

        // Delete triggers of steps that were deleted
        await Promise.all(this.triggers.filter(({ triggerID }) => {
          return !triggers.find(t => triggerID === t.triggerID)
        }).map(({ triggerID }) => {
          return this.$AutomationAPI.triggerDelete({ triggerID })
        }),
        ).then(async () => {
          await Promise.all(triggers.map(t => {
            // Update triggers that already have an ID
            if (t.triggerID) {
              return this.$AutomationAPI.triggerUpdate({
                ...t,
                workflowStepID: t.stepID,
              })
            } else {
              // Create the other triggers
              return this.$AutomationAPI.triggerCreate({
                ...t,
                workflowID: wf.workflowID,
                workflowStepID: t.stepID,
                ownedBy: this.userID,
              })
            }
          })).catch(() => {
            throw new Error('Make sure all trigger steps are properly configured')
          })

          await this.fetchTriggers(wf.workflowID)

          this.changeDetected = false
          window.onbeforeunload = null

          this.workflow = wf
          this.raiseSuccessAlert('Workflow updated')

          if (isNew) {
            // Redirect to edit route if new
            this.$router.push({ name: 'workflow.edit', params: { workflowID: this.workflow.workflowID } })
          }
        })
      } catch (e) {
        this.defaultErrorHandler('Save failed')(e)
      }
    },

    deleteWorkflow () {
      if (this.workflow.workflowID) {
        this.$AutomationAPI.workflowDelete(this.workflow)
          .then(() => {
            // Disable unsaved changes prompt
            this.workflow = {}
            this.$router.push({ name: 'workflow.list' })

            this.raiseSuccessAlert('Workflow deleted')
          })
          .catch(this.defaultErrorHandler('Failed to delete workflow'))
      }
    },
  },
}
</script>

<style lang="scss" scoped>
.editor {
  width: calc(100% - 77px);
}

.workflow-selector {
  font-size: 1rem;
  max-height: 40px;
}
</style>

<style lang="scss">
.v-select .vs__selected-options {
  flex-wrap: nowrap;
  white-space: nowrap;
  overflow: hidden;
}
</style>
