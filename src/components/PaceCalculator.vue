<script setup lang="ts">
import { ref } from 'vue';

const ranges: ('from' | 'to')[] = ['from', 'to'];

class RangeValue<T> {
  from: T;
  to: T;

  constructor(from: T, to: T) {
    this.from = from;
    this.to = to;
  }
}

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

  const totalKilometers = new RangeValue(0, 0);
  const totalSeconds = new RangeValue(0, 0);

  for (const side of ranges) {
    for (const calc of calculators.value) {
      if (calc.distance[side] > 0 && calc.distance[side] < Infinity) {
        totalKilometers[side] += calc.distance[side];
        totalSeconds[side] += calc.seconds;
      }
    }

    totalDistance.value[side] = formatNumber(totalKilometers[side]);
    totalKilometersPerHour.value[side] = formatNumber(totalKilometers[side] / (totalSeconds[side] / 3600));
  }
}

class PaceCalculatorItem {
  repeats: number = 1;

  distance = new RangeValue(0, 0);
  pace = new RangeValue(0, 0);
  kilometersPerHour = new RangeValue(0, 0);
  inputPace = new RangeValue('', '');

  seconds = 0
  inputTime = ''

  calculate() {
    this.seconds = stringToSeconds(this.inputTime);
    if (this.seconds) {
      this.seconds *= this.repeats;
    }

    for (const side of ranges) {
      this.pace[side] = stringToSeconds(this.inputPace[side]);

      this.kilometersPerHour[side] = calculateKmPerHour(this.pace[side] || 0);
      this.distance[side] = ((1000 / this.pace[side]) * this.seconds) / 1000 || 0;
    }
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

  // Find the general indent level
  const indentLevel = lines.reduce((minIndent, line) => {
    const match = line.match(/^(\s*)\S/);
    if (match) {
      return Math.min(minIndent, match[1].length);
    }
    return minIndent;
  }, Infinity);

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

    // If the line is prefixed with a number of spaces larger than 'indentLevel',
    // it's considered a sub-item and will be repeated if numberOfRepeats > 1
    if (match[1].length > indentLevel && numberOfRepeats > 1) {
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

      calcItem.inputPace.from = secondsToString(minPace, 2)
      calcItem.inputPace.to = secondsToString(maxPace, 2)
    } else {
      calcItem.inputPace.from = secondsToString(stringToSeconds(match[4]), 2)
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

function anyCalculatorHasRange() {
  for (const calc of calculators.value) {
    if (calc.distance.to > 0 && calc.distance.to < Infinity) {
      return true;
    }
  }

  return false;
}

const calculators = ref([new PaceCalculatorItem()]);
const totalDistance = ref(new RangeValue(formatNumber(Infinity), formatNumber(Infinity)));
const totalKilometersPerHour = ref(new RangeValue(formatNumber(Infinity), formatNumber(Infinity)));
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
            <input type="text" class="input-time" placeholder="hh:mm:ss" v-model="calc.inputTime"
              @keyup="calculateTotal(calc)" />
          </td>
          <td>
            <input type="text" class="input-pace" placeholder="mm:ss" v-model="calc.inputPace.from"
              @keyup="calculateTotal(calc)" />
            &ndash; <input type="text" class="input-pace" placeholder="mm:ss" v-model="calc.inputPace.to"
              @keyup="calculateTotal(calc)" />
          </td>
          <td>
            <span>x{{ calc.repeats }}</span>&nbsp;
            <button @click="incrRepeats(calc)">&plus;</button>
            <button @click="decrRepeats(calc)" :disabled="calc.repeats == 1">&minus;</button>
          </td>
          <td>
            =
            <template v-if="calc.distance.to > 0 && calc.distance.to < Infinity">
              <span class="calculated-distance">
                {{ formatNumber(calc.distance.to) }}&ndash;{{ formatNumber(calc.distance.from) }} km
                ({{ formatNumber(calc.kilometersPerHour.to) }}&ndash;{{ formatNumber(calc.kilometersPerHour.from) }}
                km/h)</span>
            </template>
            <template v-else>
              <span class="calculated-distance">{{ formatNumber(calc.distance.from) }} km ({{
                formatNumber(calc.kilometersPerHour.from)
              }}
                km/h)</span>
            </template>
          </td>
        </tr>
      </tbody>
      <tfoot v-if="calculators.length > 1">
        <tr>
          <td colspan="4"></td>
          <td>
            =
            <template v-if="anyCalculatorHasRange()">
              <span class="calculated-distance">{{ totalDistance.to }}&ndash;{{ totalDistance.from }} km (
                {{ totalKilometersPerHour.to }}&ndash;{{ totalKilometersPerHour.from }} km/h avg.)</span>
            </template>
            <template v-else>
              <span class="calculated-distance">{{ totalDistance.from }} km ({{ totalKilometersPerHour.from }} km/h avg.)</span>
            </template>
          </td>
        </tr>
      </tfoot>
    </table>
  </div>
  <div>
    <h3>Import Workout from TrainingPeaks</h3>
    <p>
    <textarea id="txt-workout-import">
# Example workout for testing
Warm up
15 min @ 05:44 min/km
Zone 2

Warm up
20:30 min @ 05:44-06:33 min/km
Zone 1-Zone 2

    Repeat 15 times

      &nbsp;&nbsp;&nbsp;&nbsp;Hard
      &nbsp;&nbsp;&nbsp;&nbsp;30 sec @ 03:59-04:22 min/km
      &nbsp;&nbsp;&nbsp;&nbsp;Zone 5b-Zone 5c

      &nbsp;&nbsp;&nbsp;&nbsp;Easy
      &nbsp;&nbsp;&nbsp;&nbsp;30 sec @ 05:24-06:07 min/km
      &nbsp;&nbsp;&nbsp;&nbsp;Zone 1-Zone 2

Cool Down
10 min @ 05:44-06:33 min/km
Zone 1-Zone 2

    Repeat 7 times

      &nbsp;&nbsp;&nbsp;&nbsp;Hard
      &nbsp;&nbsp;&nbsp;&nbsp;1:30 min @ 03:59-04:22 min/km
      &nbsp;&nbsp;&nbsp;&nbsp;Zone 5b-Zone 5c

      &nbsp;&nbsp;&nbsp;&nbsp;Easy
      &nbsp;&nbsp;&nbsp;&nbsp;30 sec @ 05:24-06:07 min/km
      &nbsp;&nbsp;&nbsp;&nbsp;Zone 1-Zone 2

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
  width: 20%;
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
