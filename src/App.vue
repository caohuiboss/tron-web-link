<template>
  <div class="App">
    <TRX />
  </div>
</template>
<script setup>
import TRX from "./components/TRX.vue";
</script>
<style>
.App {
  width: 100%;
  min-height: 100vh;
  background-color: #eee;
  background-image: url("/images/pc.png");
  background-size: cover;
  background-repeat: no-repeat;
  padding-bottom: 20px;
}

@media screen and (max-width: 768px) {
  .App {
    width: 100%;
    min-height: 100vh;
    background-color: #eee;
    background-image: url("/images/mobile_bg.png");
    background-size: cover;
    background-repeat: no-repeat;
    padding-bottom: 20px;
  }
}

</style>
