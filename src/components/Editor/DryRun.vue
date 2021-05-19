<template>
  <b-modal
    id="dry-run"
    v-model="show"
    size="lg"
    title="Initial scope"
    scrollable
    :body-class="lookup ? '' : 'p-1'"
    :ok-only="lookup"
    :ok-title="`${lookup ? 'Load and Configure' : 'Run Workflow'}`"
    cancel-title="Back"
    ok-variant="success"
    @cancel.prevent="lookup = true"
    @ok.prevent="dryRunOk()"
  >
    <div
      v-if="lookup"
    >
      <small>
        The initial scope gets injected into the workflow at execution. To load avaliable variables, input the related IDs/Handles below<br>
        If you do not wish to load any variables, click "Configure" to modify the initial scope before running workflow<br>
        Variables that can't be loaded will be auto initialized as empty
        <br><br>
        WARNING: If prompts are used in the workflow, make sure that the related webapp is also opened. Otherwise workflow will timeout
      </small>
      <div
        v-for="(p, index) in Object.values(initialScope)"
        :key="index"
        class="mt-4"
      >
        <b-form-group
          v-if="p.lookup"
          :label="p.label"
          :description="p.description"
        >
          <b-form-input
            v-model="p.value"
          />
        </b-form-group>
      </div>
    </div>

    <div
      v-else
      class="h-100"
    >
      <vue-json-editor
        :value="input"
        :options="{ name: 'Initial Scope' }"
        class="h-100"
        @input="inputEdited = $event"
      />
    </div>
  </b-modal>
</template>

<script>
import VueJsonEditor from 'v-jsoneditor'
import { encodeInput } from '../../lib/editor/dryRun'

const lookupableTypes = [
  'record',
  'oldRecord',
  'module',
  'oldModule',
  'page',
  'oldPage',
  'namespace',
  'oldNamespace',
  'user',
  'oldUser',
  'role',
  'oldRole',
  'application',
  'oldApplication',
]

export default {
  components: {
    VueJsonEditor,
  },

  props: {
    eventScope: {
      type: Array,
      default: () => [],
    },
  },

  data () {
    return {
      show: false,
      lookup: false,
      processing: false,

      initialScope: {},

      input: {},
      inputEdited: {},
    }
  },

  watch: {
    eventScope: {
      handler (eventScope) {
        this.loadScope(eventScope)
      },
    },
  },

  methods: {
    async loadScope (eventScope = []) {
      // Flag to check if lookup should be opened, or JSON editor
      let lookup = false
      if (eventScope.length) {
        this.initialScope = eventScope.reduce((initialScope, p) => {
          let label = `${p.name}${lookupableTypes.includes(p.name) ? ' (ID)' : ''}`
          if (p.type === 'ComposeNamespace' || p.type === 'ComposeModule') {
            label = `${p.name} (Handle)`
          }

          let description = ''
          if (p.type === 'ComposeRecord') {
            description = 'Namespace and Module required'
          } else if (p.type === 'ComposeModule' || p.name === 'page' || p.name === 'oldPage') {
            description = 'Namespace required'
          }

          initialScope[p.name] = ({
            label,
            value: (this.initialScope[p.name] || {}).value,
            lookup: lookupableTypes.includes(p.name),
            description,
          })

          lookup = lookup ? true : lookupableTypes.includes(p.name)
          return initialScope
        }, {})

        // Set initial values for unlookable types
        encodeInput(this.initialScope, this.$ComposeAPI, this.$SystemAPI)
          .then(input => {
            this.input = input
            this.lookup = lookup
            this.show = true
          })
          .catch(this.defaultErrorHandler('Failed to load initial scope'))
      } else {
        // If no constraints, just run
        this.initialScope = {}
        this.$emit('test')
      }
    },

    async dryRunOk () {
      if (this.lookup) {
        // Lookup based on provided ids
        encodeInput(this.initialScope, this.$ComposeAPI, this.$SystemAPI)
          .then(input => {
            this.input = input
            this.inputEdited = input
            this.lookup = false
          })
          .catch(this.defaultErrorHandler('Failed to load initial scope'))
      } else {
        this.$emit('test', this.inputEdited)
        this.show = false
      }
    },
  },
}
</script>
