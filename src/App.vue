<script setup lang="ts">
import PaceCalculator from './components/PaceCalculator.vue';
import PaceConversionChart from './components/PaceConversionChart.vue';
</script>

<template>
  <main>
    <PaceCalculator />
    <PaceConversionChart
      minPace="8:30"
      maxPace="2:30"
      paceIncrement="5"
      paceZones="Z1=6:21,Z2=5:28,Z3=4:54,Z4=4:36,Z5=4:00"
    />
  </main>
</template>

<style scoped>
header {
  line-height: 1.5;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }

  .logo {
    margin: 0 2rem 0 0;
  }

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }
}
</style>
