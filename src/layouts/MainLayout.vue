<template>
  <q-layout view="hHh lpR fFf">

    <q-header elevated class="bg-primary text-white">
      <q-toolbar>
        <q-toolbar-title>
          <div style="text-align: center;">
          <q-avatar square>
            <img src="../images/JK_L_transparent_green-1024x1024.png">
          </q-avatar>
          </div>
        </q-toolbar-title>
      </q-toolbar>
    </q-header>

    <q-page-container>
      <router-view />
    </q-page-container>

    <q-footer elevated class="bg-primary text-white">
      <q-tabs>
        <q-route-tab to="/pins" exact name="pins" icon="bolt" label="Pins" />
        <q-route-tab to="/dashboard" exact name="dashboard" icon="dashboard" label="dashboard" />
        <q-route-tab to="/jobs" exact name="jobs" icon="schedule" label="jobs" />
      </q-tabs>
    </q-footer>

  </q-layout>
</template>
