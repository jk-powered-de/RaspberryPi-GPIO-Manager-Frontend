<template>
  <div class="fullscreen bg-black text-green text-center q-pa-md flex flex-center">
    <div>
      <div style="font-size: 30vh">
        404
      </div>

      <div class="text-h2" style="opacity:.4">
        Oops. Nothing here...
      </div>

      <q-btn
        class="q-mt-xl"
        color="black"
        text-color="green"
        unelevated
        to="/"
        label="Go to Dashboard"
        no-caps
      />
    </div>
  </div>
</template>

<script >
export default {
  name: 'Error404'
}
</script>
