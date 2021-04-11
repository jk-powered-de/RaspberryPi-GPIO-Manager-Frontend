<template>
  <div class="q-pa-md">
    <q-table
      title="Pins"
      :data="namedPins"
      :columns="columns"
      row-key="pin"
      :pagination="pagination"
      :hide-pagination="true"
      :loading="loadingTable">
      <template v-slot:body="props">
        <q-tr :props="props">
          <q-td key="name" :props="props">
            {{ props.row.name }}
          </q-td>
          <q-td key="pin" :props="props">
            {{ props.row.pin }}
          </q-td>
          <q-td key="state" :props="props">
            <q-btn v-if="props.row.state === 'on'" outline round color="green" icon="power_settings_new" :size="'sm'"
                   @click="turnOffNamedPin(props.row.ID)" :loading="namedPinLoading[props.row.ID]"/>
            <q-btn v-else round color="red" icon="power_settings_new" :size="'sm'"
                   @click="turnOnNamedPin(props.row.ID)" :loading="namedPinLoading[props.row.ID]"/>
          </q-td>
          <q-td key="buttons" :props="props">
            <q-btn color="primary" icon="create" :size="'xs'" @click="openForm(props.row)" outline/>
            <q-btn color="red" icon="delete_outline" :size="'xs'" class="q-ml-sm" @click="removeNamedPin(props.row.ID)"
                   :loading="deleteNamedPinLoading[props.row.ID]" outline/>
          </q-td>
        </q-tr>
      </template>
    </q-table>
    <q-page-sticky position="bottom-right" :offset="[18, 18]">
      <q-btn fab icon="add" color="primary" @click="openForm()"/>
    </q-page-sticky>

    <q-dialog v-model="namedPinForm" persistent>
      <q-card style="width: 700px; max-width: 80vw;">
        <q-bar>
          <div v-if="formContent === null || formContent === undefined ">create Named Pin</div>
          <div v-else>edit Named Pin</div>
          <q-space/>
          <q-btn dense flat icon="close" v-close-popup>
          </q-btn>
        </q-bar>

        <q-card-section>
          <q-form
            @submit="submitForm()"
            @reset="resetForm()"
            class="q-gutter-md"
            ref="namedPinForm"
          >
            <q-input
              filled
              v-model="name"
              label="Name"
              lazy-rules
              :rules="[ val => val && val.length > 0 || 'The Name is empty!',
              val => !isNameValid || 'The Name already exist']"
            />

            <q-input
              filled
              type="number"
              v-model.number="pin"
              label="Pin"
              lazy-rules
              :rules="[
          val => val !== null && val !== '' || 'The Pin is empty!',
          val => val > 1 && val < 28 || 'Pin is not available',
          val => !isPinValid || 'Pin is already named'
        ]"
            />
            <div>
              <p>Rasbperry Pi 4B</p>
              <v-zoomer style="width: 100%; height: 100%;">
                <img
                  src="../images/GPIO-Pinout-Diagram-2.png"
                  style="object-fit: contain; width: 100%; height: 100%;"
                >
              </v-zoomer>
            </div>
            <div>
              <q-btn label="Save" type="submit" color="primary" class="full-width" :loading="loadingForm"/>
            </div>
          </q-form>
        </q-card-section>
      </q-card>
    </q-dialog>
  </div>
</template>

<script>
import Vue from 'vue'
import VueZoomer from 'vue-zoomer'

Vue.use(VueZoomer)

export default {
  data () {
    return {
      loadingForm: false,
      loadingTable: false,
      serviceDomain: 'http://raspberrypi:8080/',
      namedPinForm: false,
      formContent: null,
      name: null,
      pin: null,
      pagination: {
        page: 1,
        rowsPerPage: 0
      },
      columns: [
        {
          name: 'name',
          required: true,
          label: 'Name',
          align: 'left',
          field: row => row.name,
          format: val => `${val}`,
          sortable: true
        },
        {
          name: 'pin',
          required: true,
          align: 'center',
          label: 'Pin',
          field: 'pin',
          sortable: true
        },
        {
          name: 'state',
          required: true,
          align: 'center',
          label: 'Status',
          field: 'state',
          sortable: true
        },
        {
          name: 'buttons',
          align: 'left',
          sortable: false
        }
      ],
      namedPins: [],
      namedPinLoading: [],
      deleteNamedPinLoading: [],
      refreshData: null
    }
  },
  computed: {
    isPinValid () {
      if ((this.formContent === undefined || this.formContent === null) && this.namedPins.length > 0) {
        return !!this.namedPins.find(entry => entry.pin === this.pin)
      } else {
        return !!this.namedPins.find(entry => entry.pin === this.pin && this.formContent.pin !== entry.pin)
      }
    },
    isNameValid () {
      if ((this.formContent === undefined || this.formContent === null) && this.namedPins.length > 0) {
        return !!this.namedPins.find(entry => entry.name === this.name)
      } else {
        return !!this.namedPins.find(entry => entry.name === this.name && this.formContent.name !== entry.name)
      }
    }
  },
  created: function () {
    if (process.env.DEV) {
      this.serviceDomain = 'http://localhost:8080/'
    }
    this.getNamedPins()
    this.refreshData = setInterval(this.getNamedPins, 5000)
  },
  beforeMount () {
    this.namedPins.forEach((item, index) => {
      this.$set(this.namedPinLoading, index, false)
      this.$set(this.deleteNamedPinLoading, index, false)
    })
  },
  beforeDestroy () {
    clearInterval(this.refreshData)
  },
  methods: {
    openForm (formContent) {
      if (formContent !== undefined && formContent !== null) {
        this.name = formContent.name
        this.pin = formContent.pin
      } else {
        this.name = null
        this.pin = null
      }
      this.formContent = formContent
      this.loadingForm = false
      this.namedPinForm = true
    },
    submitForm () {
      this.$refs.namedPinForm.validate().then(success => {
        if (success) {
          if (this.formContent === undefined || this.formContent === null) {
            this.addNamedPin(this.name, this.pin)
          } else {
            this.editNamedPin(this.formContent.ID, this.name, this.pin)
          }
        }
      })
    },
    resetForm () {
      this.$refs.namedPinForm.resetValidation()
      this.name = null
      this.pin = null
      this.id = null
    },
    getNamedPins () {
      this.loadingTable = true
      this.$axios.get(this.serviceDomain + 'v1/named-pins')
        .then((response) => {
          this.namedPins = response.data.data
          this.loadingTable = false
        })
        .catch((error) => {
          this.showNotify('negative', 'Could not retrieve named Pins.', error.toString())
          this.loadingTable = false
        })
    },
    addNamedPin (name, pin) {
      this.loadingForm = true
      this.$axios.post(this.serviceDomain + 'v1/named-pins', {
        name: name,
        pin: pin
      })
        .then(() => {
          this.showNotify('positive', 'Named Pin created')
          this.getNamedPins()
          this.loadingForm = false
          this.namedPinForm = false
        })
        .catch((error) => {
          this.showNotify('negative', 'Could not create Named Pin', error.toString())
          this.loadingForm = false
        })
    },
    editNamedPin (id, name, pin) {
      this.loadingForm = true
      this.$axios.patch(this.serviceDomain + 'v1/named-pins/' + id, {
        name: name,
        pin: pin
      })
        .then(() => {
          this.showNotify('positive', 'Updated Named Pin')
          this.getNamedPins()
          this.loadingForm = false
          this.namedPinForm = false
        })
        .catch((error) => {
          this.showNotify('negative', 'Could not update Named Pin', error.toString())
          this.loadingForm = false
        })
    },
    removeNamedPin (id) {
      this.$set(this.deleteNamedPinLoading, id, true)
      this.$axios.delete(this.serviceDomain + 'v1/named-pins/' + id)
        .then(() => {
          this.showNotify('positive', 'Removed Named Pin')
          this.getNamedPins()

          this.$set(this.namedPinLoading, id, false)
        })
        .catch((error) => {
          this.showNotify('negative', 'Could not remove Named Pin', error.toString())

          this.$set(this.namedPinLoading, id, false)
        })
    },
    turnOnNamedPin (id) {
      this.$set(this.namedPinLoading, id, true)
      this.$axios.post(this.serviceDomain + 'v1/named-pin/ ' + id + '/action/turnon', null)
        .then(() => {
          this.showNotify('positive', 'Turned on Named Pin')
          this.getNamedPins()

          this.$set(this.namedPinLoading, id, false)
        })
        .catch((error) => {
          this.showNotify('negative', 'Could not turn on Named Pin', error.toString())

          this.$set(this.namedPinLoading, id, false)
        })
    },
    turnOffNamedPin (id) {
      this.$axios.post(this.serviceDomain + 'v1/named-pin/ ' + id + '/action/turnoff', null)
        .then(() => {
          this.showNotify('positive', 'Turned off Named Pin')
          this.getNamedPins()
        })
        .catch((error) => {
          this.showNotify('negative', 'Could not turn off Named Pin', error.toString())
        })
    },
    showNotify (type, message, caption) {
      this.$q.notify({
        progress: true,
        type: type,
        message: message,
        caption: caption,
        timeout: 3000,
        position: 'top-right'
      })
    }
  }
}
</script>
