<template>
    <FormGroup :field=field :error=error>
        <b-form-checkbox-group
            id="`field-${field.name}`"
            v-model="value"
            :options="options"
            buttons
            stacked
            v-on:input="onInput" 
            :state="isValid" 
        ></b-form-checkbox-group>
    </FormGroup>
</template>
<script>
import FormGroup from '@/components/FormGroup.vue'
import ValidateField from '@/mixins/ValidateField.vue'

export default {
    name: 'MultiSelectField',
    mixins: [ValidateField],
    components: {
        FormGroup
    },
    computed: {
        options() {
            return this.field.multiEnumList
        }
    }
}
</script>
