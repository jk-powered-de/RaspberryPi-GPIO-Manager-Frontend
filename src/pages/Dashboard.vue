<template>
  <div class="row justify-center">
    <q-card class="my-card col-4" style="margin: 10px">
      <q-card-section>
        <div class="text-subtitle2">Active Pins</div>
      </q-card-section>

      <q-separator />

      <q-card-actions vertical v-for="namedPin of namedPins" :key="namedPin.ID">
        <q-btn flat to="/pins" v-if="namedPin.state === 'on'">{{ namedPin.name }}</q-btn>
      </q-card-actions>
    </q-card>
    <q-card class="my-card col-4" style="margin: 10px">
      <q-card-section>
        <div class="text-subtitle2">Date & Time</div>
        <div class="text-h6" v-if="datenow">{{ datenow }}</div>
        <q-circular-progress v-else
                             indeterminate
                             size="25px"
                             color="green"
                             class="q-ma-md"
        />
      </q-card-section>
    </q-card>
    <q-card class="my-card col-4" style="margin: 10px">
      <q-card-section>
        <div class="text-subtitle2">Active Jobs</div>
      </q-card-section>

      <q-separator />

      <q-card-actions vertical>
        <q-btn flat to="/jobs">{{ jobs.length }}</q-btn>
      </q-card-actions>
    </q-card>
    <q-card class="my-card col-4" style="margin: 10px">
      <q-card-section>
        <div class="text-subtitle2">Version</div>
      </q-card-section>

      <q-separator />

      <q-card-actions vertical>
        <q-btn flat @click="versionForm = true">1.0.0 Alpha</q-btn>
      </q-card-actions>
    </q-card>
    <q-dialog v-model="versionForm">
      <q-card>
        <q-card-section class="row items-center q-pb-none">
          <div class="text-h6">V.1.0.0 Alpha - Changes</div>
          <q-space />
          <q-btn icon="close" flat round dense v-close-popup />
        </q-card-section>

        <q-card-section>
          + Pin Interface <br>
          + Job Interface <br>
          + Dashboard
        </q-card-section>
      </q-card>
    </q-dialog>
  </div></template>

<script>
import moment from 'moment'

export default {
  name: 'Dashboard',
  data: function () {
    return {
      versionForm: false,
      datenow: null,
      refreshTime: null,
      jobs: [],
      namedPins: [],
      refreshData: null,
      serviceDomain: 'http://raspberrypi:8080/'
    }
  },
  created: function () {
    if (process.env.DEV) {
      this.serviceDomain = 'http://localhost:8080/'
    }
    this.getDateNow()
    this.refreshTime = setInterval(this.getDateNow, 1000)
    this.getJobsAndNamedPins()
    this.refreshData = setInterval(this.getJobsAndNamedPins, 5000)
  },
  methods: {
    getDateNow () {
      this.datenow = moment().format('HH:mm - DD.MM.YYYY')
    },
    async getJobsAndNamedPins () {
      const jobs = []

      const jobsUndone = await this.getJobsUndone()

      if (jobsUndone !== undefined) {
        for (const jobUndone of jobsUndone) {
          jobs.push(jobUndone)
        }
      }

      this.namedPins = await this.getNamedPins()

      this.jobs = jobs
    },
    getJobsUndone () {
      return this.$axios.get(this.serviceDomain + 'v1/jobs/undone')
        .then((response) => response.data.data)
        .catch((error) => {
          this.showNotify('negative', 'Could not retrieve jobs.', error.toString())
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
  },
  beforeDestroy () {
    clearInterval(this.refreshData)
    clearInterval(this.refreshTime)
  }
}
</script>
