<template>
    <div>
      <div v-if="lines.length > 0" class="my-4">
        Enregistré !
        <b-form @submit.prevent="reloadPage" novalidate class="my-4">
            <div ref="container"></div>
            <b-button type="submit" variant="primary" :disabled="!hasValues">Autre ligne</b-button>
        </b-form>
      </div>


      <div v-if="schema && schema.location && schema.location === true">
        <Map :lon="lon" :lat="lat" v-on:childToParent="onMarkerMoved"></Map>
      </div>
      <div>
        <br /><br />
        <input type="file" @change="onFileChanged">
        <b-form @submit.prevent="submit" v-show="addingLine" novalidate class="my-4">
            <div ref="container"></div>
            <b-button type="submit" variant="primary" :disabled="!hasValues">Valider la ligne</b-button>
        </b-form>
      </div>


    </div>
</template>

<script>
import Vue from 'vue'
import StringField from '@/components/StringField.vue'
import SelectField from '@/components/SelectField.vue'
import MultiSelectField from '@/components/MultiSelectField.vue'
import RadioField from '@/components/RadioField.vue'
import { EventBus } from '@/event-bus.js';
import Map from '@/components/Map.vue';

const VALIDATA_API_URL = process.env.VUE_APP_VALIDATA_API_URL
const BACKEND_URL = process.env.VUE_APP_BACKEND_URL

export default {
  name: 'schemaForm',
  components: {
      Map,
  },
  props: {
      schemaMeta: Object,
      schemaName: String,
  },
  data() {
      return {
          schema: {},
          errors: {},
          values: {},
          fieldNames: [],
          faultyFields: [],
          lines: [],
          formValidated: false,
          addingLine: true,
          hasValues: false,
          fieldNodes: [],
          lat: 48.853,
          lon: 2.35,
          selectedFile: null
      }
  },
  watch: {
      schemaMeta() {
          // executed every time a new schema is choosen, except the first time
          // reset everything (what a mess!)
          this.removeFieldNodes()
          this.lines = []
          this.addingLine = true
          this.fieldNames = []
          this.formValidated = false
          this.hasValues = false
          this.values = {}
          this.errors = {}
          this.faultyFields = []
          // launch a new form build
          this.buildForm()
          this.getposition()
      }
  },
  mounted() {
      this.buildForm()
      this.getposition()
      EventBus.$on('field-value-changed', (field, value) => {
          this.values[field] = value
          this.computeHasValues()
      })
  },
  computed: {
      filename() {
          let date = new Date()
          let name = [
              this.schemaName,
              date.toISOString()
          ].join('_')
          return `${name}.csv`
      },
      fields() {
          return [...this.fieldNames.map(f => {
              return {
                  key: f,
                  label: f,
              }
          }), {key: 'actions', label: ''}]
      },
      csvLink() {
          let csv = this.buildFullCsvContent();
          // Forcing UTF-8 encoding. See https://stackoverflow.com/questions/17879198
          let data = new Blob(["\uFEFF" + csv], {type: 'text/csv'});
          return window.URL.createObjectURL(data);
      },
      items() {
          return this.lines.map((line) => {
              let obj = {}
              this.fieldNames.forEach((field, idx) => {
                  obj[field] = line[idx]
              })
              return obj
          })
      }
  },
  methods: {
      buildForm() {
          let loader = this.$loading.show();
          fetch(this.schemaMeta.schema_url).then(r => {
              return r.json()
          }).then(data => {
              this.schema = data
              this.schema.fields.forEach((field) => {
                  this.fieldNames.push(field.name)
                  this.fieldNodes.push(this.addField(field))
              })
          }).finally(() => {
              loader.hide()
          })
      },
      // in a method because of {} binding not allowed
      computeHasValues() {
          this.hasValues = Object.keys(this.values).length > 0 && Object.values(this.values).some(v => v !== '')
      },
      buildHeaderLine() {
          return this.fieldNames.map(v => `"${v}"`).join(',')
      },
      getCurrentLine() {
          return this.fieldNames.map(f => {
              return this.values[f] || ''
          })
      },
      buildLine(line) {
          return line.map(v => `"${v}"`).join(',')
      },
      buildCurrentCsvContent() {
          return [
              this.buildHeaderLine(),
              this.buildLine(this.getCurrentLine()),
          ].join('\r\n')
      },
      buildFullCsvContent() {
          let lines = this.lines.map(l => {
              return this.buildLine(l)
          })
          return [this.buildHeaderLine(), ...lines].join('\r\n')
      },
      buildFormData() {
          let formData = new FormData()
          // Forcing UTF-8 encoding. See https://stackoverflow.com/questions/17879198
          var blob = new Blob(["\uFEFF" + this.buildCurrentCsvContent()], {type: 'text/csv'})
          formData.append('file', blob, 'data.csv')
          formData.append('schema', this.schemaMeta.schema_url)
          return formData
      },
      removeFieldNodes() {
          this.fieldNodes.forEach((child) => {
              this.$refs.container.removeChild(child)
          })
          this.fieldNodes = []
      },
      addField(field) {
          const hasEnum = field.constraints && field.constraints.enum
          const hasMultiEnum = field.multiEnum && field.multiEnumList
          const isBoolean = field.type === "boolean"

          const factory = (klass, field) => {
            const className = Vue.extend(klass)
            let instance = new className({propsData: { field }})
            instance.$mount()
            return this.$refs.container.appendChild(instance.$el)
          }

          if (hasEnum) {
            return factory(SelectField, field)
          } else if (isBoolean) {
            return factory(RadioField, field)
          } else if (hasMultiEnum){
              return factory(MultiSelectField,field)
          }
          return factory(StringField, field)
      },
      dispatchError(error) {
          let index = error['column-number']
          this.faultyFields.push(this.fieldNames[index-1])
          EventBus.$emit('field-error', this.fieldNames[index-1], error)
      },
      dispatchNoError() {
          this.fieldNames.forEach((field) => {
              if (this.faultyFields.indexOf(field) === -1) {
                  EventBus.$emit('field-no-error', field)
              }
          })
      },
      dispatchFormValidated() {
          EventBus.$emit('form-validated')
      },
      dispatchReset() {
          EventBus.$emit('form-reset')
      },
      submit() {
          let loader = this.$loading.show();

          if(this.selectedFile != null){
                
            const rndstring = this.rndStr(20)
            const formData = new FormData()
            formData.append('myFile', this.selectedFile, rndstring)


            const requestOptionsFile = {
                method: "POST",
                body: formData
            };

            fetch(`${BACKEND_URL}/file`, requestOptionsFile)
                .then(response => response.json())


            const requestOptions = {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({"schema-name":this.schema.name,"values":this.values,"fileName":this.selectedFile.name,"fileId":rndstring})
            };
            
            fetch(`${BACKEND_URL}/form`, requestOptions)
                .then(response => response.json())

          }else{
              const requestOptions = {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({"schema-name":this.schema.name,"values":this.values})
            };
            
            fetch(`${BACKEND_URL}/form`, requestOptions)
                .then(response => response.json())

          }


          fetch(`${VALIDATA_API_URL}/validate`, {
              method: 'POST',
              body: this.buildFormData()
          })
          .then(r => r.json())
          .then(data => {
              this.formValidated = true
              this.faultyFields = []
              const errors = data.report.tables[0].errors
              if (errors && errors.length > 0) {
                  errors.forEach((error) => {
                      this.dispatchError(error)
                  })
              } else {
                  this.addingLine = false
                  this.lines.push(this.getCurrentLine())
                  this.values = {}
              }
              this.dispatchNoError()
              this.dispatchFormValidated()
          }).finally(() => {
              loader.hide()
          })


      },
      addLine() {
          this.addingLine = true
          this.formValidated = false
          this.dispatchReset()
      },
      deleteLine(idx) {
          this.lines.splice(idx, 1)
          if (this.lines.length === 0) {
              this.addLine()
          }
      },
      reloadPage(){
        window.location.reload()
      },
    
      onMarkerMoved (value) {
        this.values['latitude'] = value[0]
        this.values['longitude'] = value[1]
        this.computeHasValues()
      },
      onFileChanged (event) {
        this.selectedFile = event.target.files[0]
      },
      onUpload() {
            const formData = new FormData()
            formData.append('myFile', this.selectedFile, "oioioi")
            //axios.post('http://localhost4252/file', formData)


          const requestOptions = {
            method: "POST",
            body: formData
          };
          
          fetch("http://localhost:4242/form", requestOptions)
            .then(response => response.json())


      }, 
      rndStr(len) {
          let text = ""
          let chars = "abcdefghijklmnopqrstuvwxyz"
        
        for( let i=0; i < len; i++ ) {
            text += chars.charAt(Math.floor(Math.random() * chars.length))
        }
        return text
      },
        getposition(){
          if (navigator.geolocation) {
              navigator.geolocation.getCurrentPosition(
                  position => {
                    console.log(position.coords.longitude)
                    console.log(position.coords.latitude)
                    this.lon = position.coords.longitude;
                    this.lat = position.coords.latitude;
                  },
                  function (error) {
                      alert(error.message);
                  }, {
                      enableHighAccuracy: true
                      , timeout: 5000
                  }
              );
          } else {
              alert("Geolocation is not supported by this browser.");
          }
        }
  }
}
</script>
