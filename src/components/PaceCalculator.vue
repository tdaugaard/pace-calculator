<script setup lang="ts">
import { ref } from 'vue';

function calculateKmPerHour(pace: number) {
  const metersPerSecond = 1000 / pace;
  return metersPerSecond * 3.6;
}

function parseTimeString(str: string) {
  let str_parts: string[] = [];

  str = str.replace(/ /g, '');

  if (str.includes(':')) {
    str_parts = str.split(':').filter((v) => v != '');
  } else if (str.match(/[0-9][thms]/)) {
    // Support danish "t" for time/hour in some kind of dumb way
    str = str.replace('t', 'h');
    const matches = str.matchAll(/([0-9]+)([thms])?/g);
    const object_parts = {
      h: '0',
      m: '0',
      s: '0',
    };
    let lastSuffix: string = '';

    for (const item of matches) {
      if (!item[2]) {
        // if no suffix was set, blatantly assume it must be the next after the previous one
        // so if previous was 'm', next must be 's'.
        if (lastSuffix !== '') {
          const objectKeys = Object.keys(object_parts);
          const nextSuffixIndex = objectKeys.findIndex((v) => v === lastSuffix) + 1;
          if (nextSuffixIndex > -1 && nextSuffixIndex <= objectKeys.length) {
            item[2] = objectKeys[nextSuffixIndex];
          }
        }
      } else {
        lastSuffix = item[2];
      }

      switch (item[2]) {
        case 'h':
          object_parts.h = item[1];
          break;
        case 'm':
          object_parts.m = item[1];
          break;
        case 's':
          object_parts.s = item[1];
          break;
      }
    }

    str_parts = Object.values(object_parts);
  }

  const parts: number[] = str_parts.map((v) => parseInt(v, 10));

  // Ensure array is zero padded on the left side
  if (parts.length < 3) {
    const zeroes = Array(3 - parts.length).fill(0);
    parts.unshift(...zeroes);
  }

  return parts;
}

function stringToSeconds(str: string) {
  const parts = parseTimeString(str);
  const [hours, minutes, seconds] = parts;

  return hours * 3600 + minutes * 60 + seconds;
}

function formatNumber(num: number) {
  return new Intl.NumberFormat('en-US', {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  }).format(num);
}

function recalculate() {
  const seconds = stringToSeconds(calculatorTime.value);
  const pace = stringToSeconds(calculatorPace.value);
  const distance = ((1000 / pace) * seconds) / 1000;

  calculatedKilometersPerHour.value = calculateKmPerHour(pace);

  calculatedDistance.value = isNaN(distance) ? 0 : distance;
}

const calculatedDistance = ref(0);
const calculatedKilometersPerHour = ref(0);
const calculatorTime = ref('');
const calculatorPace = ref('');
</script>

<template>
  <h1>Distance Calculator</h1>
  <div>
    <table>
      <tbody>
        <tr>
          <td>
            <p>Time</p>
            <input type="text" v-model="calculatorTime" @keyup="recalculate()" />
            (h:mm:ss)
          </td>
          <td>
            <p>Pace</p>
            <input type="text" v-model="calculatorPace" @keyup="recalculate()" />
            (mm:ss)
          </td>
          <td>
            <p>&nbsp;</p>
            =
            <span class="calculated-distance"
              >{{ formatNumber(calculatedDistance) }} km ({{
                formatNumber(calculatedKilometersPerHour)
              }}
              km/h)</span
            >
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<style scoped>
table,
input {
  font-family: monospace;
}

.calculated-distance {
  font-size: 1.3em;
}
</style>
