<template>
  <div class="app-container" id="app">
    <price-calculator/>
  </div>
</template>

<script>
import { PriceCalculator } from "./components/PriceCalculator/index";

export default {
  name: 'app',
  components: {
    PriceCalculator
  }
}
</script>

<style>
</style>
