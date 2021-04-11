<template>
  <div class="q-pa-md">
    <q-table
      title="Jobs"
      :data="jobs"
      :columns="columns"
      row-key="pin"
      :pagination="pagination"
      :hide-pagination="true"
      :loading="loadingTable">
      <template v-slot:body="props">
        <q-tr :props="props">
          <q-td key="name" :props="props">
            {{ getPinNameById(props.row.named_pin_id) }}
          </q-td>
          <q-td key="state" :props="props">
            <q-circular-progress
              indeterminate
              size="25px"
              color="green"
              class="q-ma-md"
              v-if="props.row.state === 'running'"
            />
            <q-circular-progress
              reverse
              :thickness="1"
              :value="100"
              size="25px"
              color="red"
              v-if="props.row.state === 'done'"
            />
            <q-circular-progress
              :value="100"
              size="25px"
              color="blue"
              v-if="props.row.state !== 'running' && props.row.state !== 'done'"
            />
          </q-td>
          <q-td key="startTime" :props="props">
            {{ unixToDateTime(props.row.start_time) }}
          </q-td>
          <q-td key="endTime" :props="props">
            {{ unixToDateTime(props.row.end_time) }}
          </q-td>
          <q-td key="buttons" :props="props">
            <q-btn color="red" icon="delete_outline" :size="'xs'" class="q-ml-sm" @click="removeJob(props.row.ID)"
                   :loading="deleteJobsLoading[props.row.ID]" outline/>
          </q-td>
        </q-tr>
      </template>
    </q-table>
    <q-page-sticky position="bottom-right" :offset="[18, 18]">
      <q-btn fab icon="add" color="primary" @click="openForm()"/>
    </q-page-sticky>

    <q-dialog v-model="jobForm" persistent>
      <q-card style="width: 700px; max-width: 80vw;">
        <q-bar>
          <div>create Job</div>
          <q-space/>
          <q-btn dense flat icon="close" v-close-popup>
          </q-btn>
        </q-bar>

        <q-card-section>
          <q-form
            @submit="submitForm()"
            @reset="resetForm()"
            class="q-gutter-md"
            ref="jobForm"
          >

            <q-select
              filled
              v-model="selectedNamedPins"
              multiple
              :options="namedPins"
              :rules="[ opt => !!opt || 'Select a Named Pin' ]"
              :option-label="opt => Object(opt) === opt && 'name' in opt ? opt.name : '- Null -'"
              use-chips
              stack-label
              label="GPIO Pins"
            />

            <q-input filled v-model="selectedDateFrom" :label="'From:'" :rules="[
              val => isDateFromValid || 'Date is not valid - DD.MM.YYYY',
              val => isDateInPast || 'Date is in past'
            ]">
              <template v-slot:append>
                <q-icon name="event" class="cursor-pointer">
                  <q-popup-proxy ref="qDateProxy" transition-show="scale" transition-hide="scale">
                    <q-date v-model="selectedDateFrom" mask="DD.MM.YYYY" >
                      <div class="row items-center justify-end">
                        <q-btn v-close-popup label="Close" color="primary" flat />
                      </div>
                    </q-date>
                  </q-popup-proxy>
                </q-icon>
              </template>
            </q-input>

            <q-input filled v-model="selectedDateTo" :label="'To:'" :rules="[
              val => isDateToValid || 'Date is not valid - DD.MM.YYYY',
              val => isDateFromBeforeTo || 'Date From is after To']">
              <template v-slot:append>
                <q-icon name="event" class="cursor-pointer">
                  <q-popup-proxy ref="qDateProxy" transition-show="scale" transition-hide="scale">
                    <q-date v-model="selectedDateTo" mask="DD.MM.YYYY">
                      <div class="row items-center justify-end">
                        <q-btn v-close-popup label="Close" color="primary" flat />
                      </div>
                    </q-date>
                  </q-popup-proxy>
                </q-icon>
              </template>
            </q-input>

            <q-input filled v-model="selectedStartTime" :rules="[
              val => isTimeValidFrom || 'Time is not valid - HH:mm'
            ]" label="Start-Time">
              <template v-slot:append>
                <q-icon name="access_time" class="cursor-pointer">
                  <q-popup-proxy transition-show="scale" transition-hide="scale">
                    <q-time
                      v-model="selectedStartTime"
                      now-btn format24h landscape>
                      <div class="row items-center justify-end">
                        <q-btn v-close-popup label="Close" color="primary" flat/>
                      </div>
                    </q-time>
                  </q-popup-proxy>
                </q-icon>
              </template>
            </q-input>
            <q-input filled v-model="selectedEndTime" :rules="[
              val => isTimeValidTo || 'Time is not valid - HH:mm',
              val => isTimeFromBeforeTo || 'Time From is after To'
            ]" label="End-Time">
              <template v-slot:append>
                <q-icon name="access_time" class="cursor-pointer">
                  <q-popup-proxy transition-show="scale" transition-hide="scale">
                    <q-time
                      v-model="selectedEndTime"
                      now-btn format24h landscape>
                      <div class="row items-center justify-end">
                        <q-btn v-close-popup label="Close" color="primary" flat/>
                      </div>
                    </q-time>
                  </q-popup-proxy>
                </q-icon>
              </template>
            </q-input>
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
import moment from 'moment'

export default {
  data () {
    return {
      selectedDateFrom: '',
      selectedDateTo: '',
      availableNamedPins: [],
      selectedNamedPins: [],
      selectedStartTime: {},
      selectedEndTime: {},
      loadingForm: false,
      loadingTable: false,
      serviceDomain: 'http://raspberrypi:8080/',
      jobForm: false,
      formContent: null,
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
          name: 'state',
          required: true,
          align: 'center',
          label: 'Status',
          field: 'state',
          sortable: true
        },
        {
          name: 'startTime',
          required: true,
          align: 'center',
          label: 'From',
          field: 'start_time',
          sortable: true
        },
        {
          name: 'endTime',
          required: true,
          align: 'center',
          label: 'To',
          field: 'end_time',
          sortable: true
        },
        {
          name: 'buttons',
          align: 'left',
          sortable: false
        }
      ],
      namedPins: [],
      jobs: [],
      deleteJobsLoading: [],
      refreshData: null
    }
  },
  computed: {
    isTimeFromBeforeTo () {
      return moment(this.selectedStartTime, 'HH:mm', true).isSameOrBefore(moment(this.selectedEndTime, 'HH:mm', true))
    },
    isTimeToBeforeFrom () {
      return moment(this.selectedEndTime, 'HH:mm', true).isSameOrBefore(moment(this.selectedStartTime, 'HH:mm', true))
    },
    isTimeValidTo () {
      return moment(this.selectedEndTime, 'HH:mm', true).isValid()
    },
    isTimeValidFrom () {
      return moment(this.selectedStartTime, 'HH:mm', true).isValid()
    },
    isDateInPast () {
      return moment(moment(this.selectedDateFrom, 'DD.MM.YYYY', true)).isSameOrAfter(moment(this.getToday(), 'DD.MM.YYYY', true))
    },
    isDateFromBeforeTo () {
      return moment(this.selectedDateFrom, 'DD.MM.YYYY', true).isSameOrBefore(moment(this.selectedDateTo, 'DD.MM.YYYY', true))
    },
    isDateFromValid () {
      return moment(this.selectedDateFrom, 'DD.MM.YYYY', true).isValid()
    },
    isDateToValid () {
      return moment(this.selectedDateTo, 'DD.MM.YYYY', true).isValid()
    },
    isPinValid () {
      if ((this.formContent === undefined || this.formContent === null) && this.namedPins.length > 0) {
        return !!this.namedPins.find(entry => entry.pin === this.pin)
      } else {
        return !!this.namedPins.find(entry => entry.pin === this.pin && this.formContent.pin !== entry.pin)
      }
    }
  },
  created: function () {
    if (process.env.DEV) {
      this.serviceDomain = 'http://localhost:8080/'
    }
    this.getJobsAndNamedPins()
    this.refreshData = setInterval(this.getJobsAndNamedPins, 5000)
  },
  beforeMount () {
    this.jobs.forEach((item, index) => {
      this.$set(this.deleteJobsLoading, index, false)
    })
  },
  beforeDestroy () {
    clearInterval(this.refreshData)
  },
  methods: {
    openForm () {
      this.resetForm()
      this.loadingForm = false
      this.jobForm = true
    },
    async getJobsAndNamedPins () {
      this.loadingTable = true
      const jobs = []

      const jobsUndone = await this.getJobsUndone()
      if (jobsUndone !== undefined) {
        for (const jobUndone of jobsUndone) {
          jobs.push(jobUndone)
        }
      }

      const jobsDone = await this.getJobsDone()
      if (jobsDone !== undefined) {
        for (const jobDone of jobsDone) {
          jobs.push(jobDone)
        }
      }

      this.namedPins = await this.getNamedPins()

      this.jobs = jobs
      this.loadingTable = false
    },
    resetForm () {
      this.selectedNamedPins = null
      this.selectedDateFrom = null
      this.selectedDateTo = null
      this.selectedEndTime = null
      this.selectedStartTime = null
    },
    submitForm () {
      if (this.selectedDateFrom && this.selectedDateTo) {
        this.$refs.jobForm.validate().then(success => {
          if (success) {
            this.loadingForm = true
            const dates = this.getDates(moment(this.selectedDateFrom, 'DD-MM-YYYY').toDate(), moment(this.selectedDateTo, 'DD-MM-YYYY').toDate())
            if (dates.length === 0) {
              dates.push((moment(this.selectedDateFrom, 'DD-MM-YYYY').toDate()))
            }

            const promises = []
            let hasError = false
            this.selectedNamedPins.forEach(namedPin => {
              dates.forEach(date => {
                const startTime = this.selectedStartTime.split(':')
                const endTime = this.selectedEndTime.split(':')

                const startDate = new Date(date.setHours(startTime[0], startTime[1]))
                const endDate = new Date(date.setHours(endTime[0], endTime[1]))

                promises.push(this.createJob(namedPin.ID, parseInt((startDate.getTime() / 1000).toFixed(0)), parseInt((endDate.getTime() / 1000).toFixed(0)))
                  .then((response) => {
                    console.log(response)
                  })
                  .catch((error) => {
                    console.log(error)
                    hasError = true
                  }))
              })
            })

            Promise.all(promises).then(() => {
              if (hasError) {
                this.jobForm = true
                this.showNotify('negative', 'Error during job creation.')
              } else {
                this.showNotify('positive', 'Jobs created')
                this.jobForm = false
              }
              this.getJobsAndNamedPins()
              this.loadingForm = false
            })
          }
        })
      }
    },
    getJobsUndone () {
      return this.$axios.get(this.serviceDomain + 'v1/jobs/undone')
        .then((response) => response.data.data)
        .catch((error) => {
          this.showNotify('negative', 'Could not retrieve jobs.', error.toString())
        })
    },
    getJobsDone () {
      return this.$axios.get(this.serviceDomain + 'v1/jobs/done', { params: { limit: 20 } })
        .then((response) => {
          return response.data.data
        })
        .catch((error) => {
          this.showNotify('negative', 'Could not retrieve jobs done.', error.toString())
        })
    },
    createJob (pinID, startTime, endTime) {
      return this.$axios.post(this.serviceDomain + 'v1/jobs', {
        named_pin_id: pinID,
        start_time: startTime,
        end_time: endTime
      })
    },
    getNamedPins () {
      return this.$axios.get(this.serviceDomain + 'v1/named-pins')
        .then((response) => {
          return response.data.data
        })
        .catch((error) => {
          this.showNotify('negative', 'Could not retrieve Named Pins.', error.toString())
        })
    },
    removeJob (id) {
      this.$set(this.deleteJobsLoading, id, true)
      this.$axios.delete(this.serviceDomain + 'v1/jobs/' + id)
        .then(() => {
          this.showNotify('positive', 'Job removed')
          this.getJobsAndNamedPins()
          this.$set(this.deleteJobsLoading, id, false)
        })
        .catch((error) => {
          this.showNotify('negative', 'Could not remove job', error.toString())
          this.$set(this.deleteJobsLoading, id, false)
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
    },
    getDates (startDate, endDate) {
      const dates = []
      let currentDate = startDate
      const addDays = function (days) {
        const date = new Date(this.valueOf())
        date.setDate(date.getDate() + days)
        return date
      }
      while (currentDate <= endDate) {
        dates.push(currentDate)
        currentDate = addDays.call(currentDate, 1)
      }
      return dates
    },
    getPinNameById (id) {
      let name = 'Error: Unknown Pin'
      this.namedPins.forEach(namedPin => {
        if (namedPin.ID === id) {
          name = namedPin.name
        }
      })
      return name
    },
    unixToDateTime (unixTimestamp) {
      const date = new Date(unixTimestamp * 1000)
      const year = date.getFullYear()
      const month = '0' + (date.getMonth() + 1)
      const day = '0' + date.getDate()
      const hours = '0' + date.getHours()
      const minutes = '0' + date.getMinutes()
      return day.substr(-2) + '.' + month.substr(-2) + '.' + year + ' ' + hours.substr(-2) + ':' + minutes.substr(-2)
    },
    getToday () {
      const today = new Date()
      const dd = String(today.getDate()).padStart(2, '0')
      const mm = String(today.getMonth() + 1).padStart(2, '0')
      const yyyy = today.getFullYear()
      return dd + '.' + mm + '.' + yyyy
    }
  }
}
// ToDo Install axios retry
</script>
