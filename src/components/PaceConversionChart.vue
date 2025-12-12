<script setup lang="ts">
import { computed, ref } from 'vue'

const tableDisplayEverything = ref(false);

const props = defineProps<{
  paceRange: string;
  visiblePaceRange: string;
  paceIncrement: string;
  paceZones: string;
}>();

function kmToMile(speed: number) {
  return speed * 0.621371192;
}

function mileToKm(speed: number) {
  return speed * 1.609344;
}

function calculateKmPerHour(pace: number) {
  const metersPerSecond = 1000 / pace;
  return metersPerSecond * 3.6;
}

function convertPaceRange(str: string) {
  const [maxStr, minStr] = str.split('-');

  return [convertPaceString(minStr), convertPaceString(maxStr)].sort((a, b) => b - a);
}

function convertPaceString(str: string) {
  const [minutes, seconds] = str.split(':');
  const sec = Math.floor(parseInt(seconds, 10) / 5) * 5;

  return parseInt(minutes, 10) * 60 + sec;
}

function formatNumber(num: number) {
  return new Intl.NumberFormat('en-US', {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  }).format(num);
}

interface SecondsToTimeStringOptions {
  hours: boolean;
  sep: string;
}

function secondsToTimeString(seconds: number, opts?: SecondsToTimeStringOptions) {
  const hours = Math.floor(seconds / 3600);
  const mins = Math.floor((seconds - hours * 3600) / 60);
  const secs = Math.floor(seconds % 60);

  const parts = [hours, mins, secs].map((v) => String(v).padStart(2, '0'));
  const parts_separators = ['h', 'm', 's'];

  if (!opts?.hours) {
    parts.shift();
    parts_separators.shift();
  }

  if (!opts?.sep || opts.sep == ':') {
    return parts.join(':');
  }

  let str = '';
  parts.forEach((v, i) => {
    str += `${v}${parts_separators[i]} `;
  });

  return str;
}

type IPaceZones = number[];

function parsePaceZones(str: string) {
  return str
    .split(',')
    .map(convertPaceString)
}

function findPaceZone(pace: number, paceZones: IPaceZones) {
  for (const [zone, zpace] of paceZones.entries()) {
    if (pace >= zpace) {
      return zone + 1;
    }
  }

  return 6;
}

function getPaceZoneLabel(paceZones: IPaceZones, paceZone: number) {
  if (paceZone === 6) {
    return `> ${secondsToTimeString(paceZones[paceZones.length - 1])}`;
  }
  if (paceZone === 1) {
    return `< ${secondsToTimeString(paceZones[0])}`;
  }

  return `${secondsToTimeString(paceZones[paceZone - 1])} - ${secondsToTimeString(paceZones[paceZone])}`;
}

class PaceTableData {
  paceZone: number;
  kmPerHour: number;
  miPerHour: number;
  minsPerKm: number;
  minsPerMi: number;
  distanceTime: number[];

  constructor(
    paceZone: number,
    kmPerHour: number,
    miPerHour: number,
    minsPerKm: number,
    minsPerMi: number,
    distanceTime: number[] = []
  ) {
    this.paceZone = paceZone
    this.kmPerHour = kmPerHour
    this.miPerHour = miPerHour
    this.minsPerKm = minsPerKm
    this.minsPerMi = minsPerMi
    this.distanceTime = distanceTime
  }
};

class PaceTableDivider {
  label: string;

  constructor(label: string) {
    this.label = label;
  }
};

type paceTableRow = PaceTableData | PaceTableDivider;

type paceTable = {
  header: string[];
  data: paceTableRow[];
};

const paceZoneLabels = [
  'Recovery',
  'Endurance',
  'Tempo',
  'Threshold',
  'VO2 Max',
  'Anaerobic',
]

// In meters
const runDistances = {
  '5K': 5000,
  '10K': 10000,
  'Half Marathon': 21975,
  'Marathon': 42195,
};

const tableData: paceTable = {
  header: ['km/h', 'mi/h', 'Mins/km', 'Mins/mi'],
  data: [],
};

Object.keys(runDistances).forEach((v) => {
  tableData.header.push(v);
});

const [paceRangeMin, paceRangeMax] = convertPaceRange(props.paceRange);
const [visiblePaceRangeMin, visiblePaceRangeMax] = convertPaceRange(props.visiblePaceRange);
const paceIncrement = parseInt(props.paceIncrement, 10);
const paceZones = parsePaceZones(props.paceZones);

function toggleDataVisibility() {
  tableDisplayEverything.value = tableDisplayEverything.value ? false : true;
  renderPaceTable();
}

function isPaceVisible(pace: number) {
  if (tableDisplayEverything.value) {
    return true;
  }
  return pace <= visiblePaceRangeMin && pace >= visiblePaceRangeMax;
}

function renderPaceTable() {
  const zoneDividersAdded = new Set<number>();

  tableData.data = [];
  for (let pace = paceRangeMin; pace >= paceRangeMax; pace -= paceIncrement) {
    if (!isPaceVisible(pace)) {
      continue;
    }

    const currentPaceZone = findPaceZone(pace, paceZones);
    if (!zoneDividersAdded.has(currentPaceZone)) {
      tableData.data.push(new PaceTableDivider(`<span class="pace-zone-${currentPaceZone}">
        Z${currentPaceZone}: ${paceZoneLabels[currentPaceZone - 1]} (${getPaceZoneLabel(paceZones, currentPaceZone)})
      </span>`));

      zoneDividersAdded.add(currentPaceZone);
    }

    const kmPerHour = calculateKmPerHour(pace);
    const paceZone = findPaceZone(pace, paceZones);
    const item = new PaceTableData(
      paceZone,
      kmPerHour,
      kmToMile(kmPerHour),
      pace,
      mileToKm(pace)
    );

    Object.values(runDistances).forEach(v => {
      item.distanceTime.push((v / 1000) * pace);
    });

    tableData.data.push(item);
  }
}

const visibilityButtonLabel = computed(() => {
  const [minPace, maxPace] = [visiblePaceRangeMin, visiblePaceRangeMax].map(v => secondsToTimeString(v));
  return tableDisplayEverything.value ? `Show only ${minPace} - ${maxPace}` : 'Show Everything';
});

renderPaceTable()
</script>

<template>
  <h1>Running Pace Conversion Chart</h1>

  <nav>
    <button @click="toggleDataVisibility" class="btn-visibility-toggle">{{ visibilityButtonLabel }}</button>
  </nav>

  <table>
    <thead>
      <th v-for="(label, index) in tableData.header" :key="index">{{ label }}</th>
    </thead>
    <tbody>
      <template v-for="(data, index) in tableData.data" :key="index">
        <tr v-if="data instanceof PaceTableDivider">
          <th :colspan="tableData.header.length" style="text-align: left; padding-top: 1em;" v-html="data.label"></th>
        </tr>
        <tr v-else-if="data instanceof PaceTableData" :class="`pace-zone-${data.paceZone}`">
          <td>{{ formatNumber(data.kmPerHour) }}</td>
          <td>{{ formatNumber(data.miPerHour) }}</td>
          <td>{{ secondsToTimeString(data.minsPerKm) }}</td>
          <td>{{ secondsToTimeString(data.minsPerMi) }}</td>
          <td v-for="(time, tindex) in data.distanceTime" :key="tindex">
            {{ secondsToTimeString(time, { hours: true, sep: 'hms' }) }}
          </td>
        </tr>
      </template>
    </tbody>
  </table>
</template>

<style scoped>
.btn-visibility-toggle {
  margin-bottom: 1em;
  padding: 0.5em 1em;
  font-size: 1em;
  width: 20vw;
}

table {
  width: 100vh;
  font-size: 1.1em;
}

th,
td {
  text-align: right;
}

th {
  text-transform: uppercase;
  font-weight: bold;
}

td {
  font-family: monospace;
}
</style>
