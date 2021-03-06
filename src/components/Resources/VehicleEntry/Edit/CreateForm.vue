<template>
  <form id="main-form" class="smart-form">
    <fieldset>
      <div class="row">
        <template v-for="field in __formFields">
          <component
          :is="field.component"
          :wrapper-class="field.wrapperClass"
          :data-vv-as="field.label"
          :select-options="field.selectOptions"
          :using-as-provider="field.usingAsProvider"
          :using-as-label="field.usingAsLabel"
          :mask="field.mask"
          :lazy="field.lazy"
          :using-as-value="field.usingAsValue"
          :has-related="field.hasRelated"
          :related="field.related"
          v-validate="$data[field['name'] + '_rule']"
          :name="field.name"
          v-model="$data[field['name']]"
          :icon="field.icon"
          :label="field.label"
          :placeholder="field.placeholder"
          :readonly="readonly"
          :custom-error="errors.first(field.name)"
          :valid="fields.valid(field.name)"
          :failed="fields.failed(field.name)"
          >
        </component>
      </template>
    </div>
  </fieldset>
</form>
</template>

<script>
  import jquery from 'jquery'
  import Api from './../Api'
  import { map, mapKeys, filter } from 'lodash'
  import {
    __fields,
    __defaultBasePath,
    __defaultFieldsDetail
  } from './../metadata'

  export default {
    props: {
      group: {
        type: Number
      },
      formEditable: {
        type: Boolean,
        default: false
      }
    },
    computed: {
      basePath () {
        return __defaultBasePath
      },
      __fields () {
        return __fields
      },
      __formFields () {
        return filter(__fields, el => el.group === this.group && typeof el.details === 'undefined' || el.details === false)
      },
      showVehiclePanel () {
        return this.year.length && this.model.length && this.brand.length && this.vehicle.length
      },
      __defaultFieldsDetail () {
        return __defaultFieldsDetail
      }
    },
    watch: {
      year (value) {
        if (value === '') {
          this.$root.bus.$emit('select-vehicle-outdated', {})
        } else {
          let details = value.split(' ')

          this.$root.bus.$emit('select-vehicle-outdated', {
            'value': value,
            query: {
              q: `year:${details[0]};brand:${this.brand};model:${this.model};fuel:${details[1]}`,
              sf: 'year:ilike;brand:ilike;model:ilike;fuel:ilike',
              _and: true
            }
          })
        }
      },
      vehicle (value) {
        if (value === '') {
          this.detailsCar = {}
        } else {
          this.$root.axios.get('vehicle-fipes/' + value).then(({data}) => {
            let vehicle = data.data
            this.detailsCar = mapKeys(vehicle, (value, key) => { return 'detail_' + key })
          })
        }
      }
    },
    created () {
      this.readonly = this.formEditable
      this.api = new Api(this, this.basePath)

      this.$root.axios.get(`${this.basePath}/rules/create`).then(({ data }) => {
        let rules = data.data.rules
        let keys = Object.keys(rules)
        keys.forEach((prop, index) => {
          this[prop + '_rule'] = rules[prop]
        })
      })
    },
    data () {
      let dynamicData = {}

      map(__fields, (field) => {
        let ruleKey = `${field.name}_rule`
        dynamicData[field.name] = ''
        dynamicData[ruleKey] = ''
      })

      return {
        id: null,
        readonly: false,
        detailsCar: {},
        ...dynamicData
      }
    },
    methods: {
      clearState () {
        this.errors.clear()
        this.fields.reset()
      },
      create () {
        this.$validator.validateAll().then(() => {
          this.api.create(this._data, ({ data }) => {
            this.$router.push(`/${this.basePath}`)
          })
        }).catch(() => {})
      },
      destroy () {
        this.api.delete(this.id, ({ data }) => {
          this.$router.push(`/${this.basePath}`)
        })
      },
      disableEdit () {
        this.fetch()
        this.readonly = true
      },
      allowEdit () {
        new Promise((resolve, reject) => {
          this.readonly = false
          resolve(this.readonly)
        }).then((readonly) => {
          !readonly ? jquery('#main-form [type=text]').first().focus() : ''
        })
      }
    }
  }
</script>
