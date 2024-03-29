<script setup lang="ts">
const props = defineProps<{
  minPace: string;
  maxPace: string;
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

type IPaceZones = Record<string, Record<string, number>>;

function parsePaceZones(str: string) {
  const zones = str
    .toLowerCase()
    .split(',')
    .map((v) => {
      const [zone, pace_str] = v.split('=');

      return {
        zone: parseInt(zone.replace(/^z/, ''), 10),
        pace: convertPaceString(pace_str),
      };
    });

  const paceZones: IPaceZones = {};

  zones.forEach((zone) => {
    paceZones[zone.zone] = zone;
  });

  return paceZones;
}

function findPaceZone(pace: number, paceZones: IPaceZones) {
  let paceZone = 6;

  for (const zone of Object.values(paceZones)) {
    if (pace >= zone.pace) {
      paceZone = zone.zone;
      break;
    }
  }

  return paceZone;
}

type paceTableData = {
  paceZone: number;
  kmPerHour: number;
  miPerHour: number;
  minsPerKm: number;
  minsPerMi: number;
  distanceTime: number[];
};
type paceTable = {
  header: string[];
  data: paceTableData[];
};

// In kilometers
const runDistances = {
  '5K': 5,
  '10K': 10,
  'Half Marathon': 21.0975,
  Marathon: 42.195,
};

const tableData: paceTable = {
  header: ['km/h', 'mi/h', 'Mins/km', 'Mins/mi'],
  data: [],
};

Object.keys(runDistances).forEach((v) => {
  tableData.header.push(v);
});

const minPaceSeconds = convertPaceString(props.minPace);
const maxPaceSeconds = convertPaceString(props.maxPace);
const paceIncrement = parseInt(props.paceIncrement, 10);
const paceZones = parsePaceZones(props.paceZones);

for (let pace = minPaceSeconds; pace >= maxPaceSeconds; pace -= paceIncrement) {
  const kmPerHour = calculateKmPerHour(pace);
  const paceZone = findPaceZone(pace, paceZones);
  const item: paceTableData = {
    paceZone,
    kmPerHour: kmPerHour,
    miPerHour: kmToMile(kmPerHour),
    minsPerKm: pace,
    minsPerMi: mileToKm(pace),
    distanceTime: [],
  };

  Object.values(runDistances).forEach((v) => {
    item.distanceTime.push(v * pace);
  });

  tableData.data.push(item);
}
</script>

<template>
  <h1>Running Pace Conversion Chart</h1>
  <table>
    <thead>
      <th v-for="(label, index) in tableData.header" :key="index">{{ label }}</th>
    </thead>
    <tbody>
      <tr
        v-for="(data, index) in tableData.data"
        :key="index"
        :class="`pace-zone-${data.paceZone}`"
      >
        <td>{{ formatNumber(data.kmPerHour) }}</td>
        <td>{{ formatNumber(data.miPerHour) }}</td>
        <td>{{ secondsToTimeString(data.minsPerKm) }}</td>
        <td>{{ secondsToTimeString(data.minsPerMi) }}</td>
        <td v-for="(time, tindex) in data.distanceTime" :key="tindex">
          {{ secondsToTimeString(time, { hours: true, sep: 'hms' }) }}
        </td>
      </tr>
    </tbody>
  </table>
</template>

<style scoped>
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

tr.pace-zone-1 {
  color: gray;
}
tr.pace-zone-2 {
  color: chartreuse;
}
tr.pace-zone-3 {
  color: bisque;
}
tr.pace-zone-4 {
  color: chocolate;
}
tr.pace-zone-5 {
  color: crimson;
}
tr.pace-zone-6 {
  color: firebrick;
}
</style>
