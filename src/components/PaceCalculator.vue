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

function secondsToString(totalSeconds: number, limitParts: number = 3) {
  const hours = Math.floor(totalSeconds / 3600);
  const minutes = Math.floor((totalSeconds % 3600) / 60);
  const seconds = Math.floor(totalSeconds % 60);

  return timePartsToString([hours, minutes, seconds].slice(-limitParts));
}

function timePartsToString(parts: number[]) {
  return parts
    .map((v) => v.toString().padStart(2, '0'))
    .join(':');
}

function formatNumber(num: number) {
  return new Intl.NumberFormat('en-US', {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  }).format(num);
}

function calculateTotal(calcItem: PaceCalculatorItem) {
  calcItem.calculate();

  let totalKilometers = 0;
  let totalSeconds = 0;

  for (const calc of calculators.value) {
    totalKilometers += calc.distance;
    totalSeconds += calc.seconds;
  }

  totalDistance.value = formatNumber(totalKilometers);
  totalKilometersPerHour.value = formatNumber(totalKilometers / (totalSeconds / 3600));
}

class PaceCalculatorItem {
  distance: number = 0;
  seconds: number = 0;
  pace: number = 0;
  repeats: number = 1;
  kilometersPerHour: number = 0;

  inputTime: string = '';
  inputPace: string = '';

  calculate() {
    this.seconds = stringToSeconds(this.inputTime);
    this.pace = stringToSeconds(this.inputPace);

    if (this.seconds) {
      this.seconds *= this.repeats;
    }

    this.kilometersPerHour = calculateKmPerHour(this.pace || 0);
    this.distance = ((1000 / this.pace) * this.seconds) / 1000 || 0;
  }
}

function addCalculator(index: number) {
  calculators.value.splice(index + 1, 0, new PaceCalculatorItem());
}

function removeCalculator(index: number) {
  if (calculators.value.length > 1) {
    calculators.value.splice(index, 1);
  }
}

function incrRepeats(calc: PaceCalculatorItem) {
  calc.repeats += 1;
  calculateTotal(calc);
}

function decrRepeats(calc: PaceCalculatorItem) {
  if (calc.repeats > 1) {
    calc.repeats -= 1;
  }
  calculateTotal(calc);
}

function importWorkout() {
  const textarea = document.getElementById('txt-workout-import') as HTMLTextAreaElement;
  if (!textarea) return;

  const lines = textarea.value.split('\n');
  const calcs: PaceCalculatorItem[] = [];
  let numberOfRepeats = 1

  let match
  for (const line of lines) {
    match = line.match(/Repeat\s+([0-9]+)\s+times/i);
    if (match) {
      numberOfRepeats = parseInt(match[1], 10);
      continue
    }

    match = line.match(/^(\s*)([0-9:]+)\s+(min|sec)\s+@\s+(?:([0-9:]+)(?:-([0-9:]+))?)\s*min\/km/i);
    if (!match) {
      continue
    }

    const calcItem = new PaceCalculatorItem();

    // If the line is prefixed with any number of spaces, it's considered a sub-item and will be repeated
    // if numberOfRepeats > 1
    if (match[1] && numberOfRepeats > 1) {
      calcItem.repeats = numberOfRepeats
    }

    let timeValue: number[] = [];
    const timeString = match[2]
    const timeUnitString = match[3]

    if (timeString.includes(':')) {
      timeValue = parseTimeString(timeString)
    } else {
      const value = parseInt(timeString, 10);
      timeValue = parseTimeString(value + (timeUnitString === 'min' ? 'm' : 's'))
    }

    calcItem.inputTime = timePartsToString(timeValue)

    if (match[5]) {
      const minPace = stringToSeconds(match[4])
      const maxPace = stringToSeconds(match[5])

      calcItem.inputPace = secondsToString((maxPace + minPace) / 2, 2)
    } else {
      calcItem.inputPace = secondsToString(stringToSeconds(match[4]), 2)
    }

    calcs.push(calcItem)
    calculateTotal(calcItem)
  }

  if (calcs.length === 0) {
    alert('No valid pace/time entries found in the workout text.')
    return
  }

  calculators.value = calcs;
  calculateTotal(calcs[0])
}

const calculators = ref([new PaceCalculatorItem()]);
const totalDistance = ref(formatNumber(Infinity));
const totalKilometersPerHour = ref(formatNumber(Infinity));
</script>

<template>
  <h1>Distance Calculator</h1>
  <div>
    <table>
      <thead>
        <tr>
          <th>&nbsp;</th>
          <th>Time</th>
          <th>Pace</th>
          <th>Repeats</th>
          <th>&nbsp;</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(calc, idx) in calculators" :key="idx">
          <td>
            <button @click="addCalculator(idx)">&plus;</button>
            <button @click="removeCalculator(idx)" :disabled="calculators.length == 1">
              &minus;
            </button>
          </td>
          <td>
            {{ idx + 1 }}.
            <input
              type="text"
              class="input-time"
              placeholder="hh:mm:ss"
              v-model="calc.inputTime"
              @keyup="calculateTotal(calc)"
            />
          </td>
          <td>
            <input
              type="text"
              class="input-pace"
              placeholder="mm:ss"
              v-model="calc.inputPace"
              @keyup="calculateTotal(calc)"
            />
          </td>
          <td>
            <span>x{{ calc.repeats }}</span
            >&nbsp;
            <button @click="incrRepeats(calc)">&plus;</button>
            <button @click="decrRepeats(calc)" :disabled="calc.repeats == 1">&minus;</button>
          </td>
          <td>
            =
            <span class="calculated-distance"
              >{{ formatNumber(calc.distance) }} km ({{
                formatNumber(calc.kilometersPerHour)
              }}
              km/h)</span
            >
          </td>
        </tr>
      </tbody>
      <tfoot v-if="calculators.length > 1">
        <tr>
          <td colspan="4"></td>
          <td>
            =
            <span class="calculated-distance">{{ totalDistance }} km ({{ totalKilometersPerHour }} km/h avg.)</span>
          </td>
        </tr>
      </tfoot>
    </table>
  </div>
  <div>
    <p>
      <h3>Import Workout from TrainingPeaks</h3>
      <textarea id="txt-workout-import">
# Example workout for testing
Warm up
15 min @ 05:44 min/km
Zone 2

Warm up
20:30 min @ 05:44-06:33 min/km
Zone 1-Zone 2

Repeat 15 times

    Hard
    30 sec @ 03:59-04:22 min/km
    Zone 5b-Zone 5c

    Easy
    30 sec @ 05:24-06:07 min/km
    Zone 1-Zone 2

Cool Down
10 min @ 05:44-06:33 min/km
Zone 1-Zone 2

Repeat 7 times

    Hard
    1:30 min @ 03:59-04:22 min/km
    Zone 5b-Zone 5c

    Easy
    30 sec @ 05:24-06:07 min/km
    Zone 1-Zone 2

Cool Down
15 min @ 05:44 min/km
Zone 2
      </textarea>
    </p>
    <button @click="importWorkout()">Import</button>
  </div>
</template>

<style scoped>
input.input-pace {
  width: 40%;
}

input.input-time {
  width: 40%;
}

table,
input {
  font-family: monospace;
}

.calculated-distance {
  font-size: 1.3em;
}

textarea {
  width: 50vw;
  height: 20vh;
}
</style>
