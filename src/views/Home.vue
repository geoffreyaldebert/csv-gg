<template>
    <div>
            <br/><br/>
            <b-form-select v-model="schemaName" :options="options">
                <template slot="first">
                  <option :value="null" disabled>Choisissez un schéma</option>
                </template>
            </b-form-select>
        </b-form-group>

        <schema-form v-if="schema" :schema-meta="schema" :schema-name="schemaName"></schema-form>
    </div>
</template>

<script>
import SchemaForm from './SchemaForm.vue'

const SCHEMAS_CATALOG_URL = process.env.VUE_APP_SCHEMAS_CATALOG_URL

export default {
  name: 'home',
  components: {SchemaForm},
  data() {
      return {
          schemaName: this.$route.query.schema,
          schemas: null,
          options: [],
      }
  },
  mounted() {
      let loader = this.$loading.show()
      fetch(`${SCHEMAS_CATALOG_URL}`).then(r => {
          return r.json()
      }).then(data => {
          this.schemas = data.schemas
          this.options = this.schemas.map(s => {
              return {value: s.name, text: s.title || s.name}
          })
      }).finally(() => {
          loader.hide()
      })
  },
  computed: {
      schema() {
          if (!this.schemaName) return
          if (!this.schemas) return
          return this.schemas.find(s => s.name === this.schemaName)
      }
  },
  watch: {
    schemaName(newVal) {
        this.$router.push({ query: { ...this.$route.query, schema: newVal } })
    }
  }
}
</script>
