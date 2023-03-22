<template>
  <div v-if="!webcodecsSupported">Use WebCodecs API supported browsers like Chrome</div>
  <ScreenShare v-if="webcodecsSupported"/>
</template>

<script setup lang="ts">
import ScreenShare from "@/components/ScreenShare.vue";

const webcodecsSupported = 'VideoEncoder' in window;
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
