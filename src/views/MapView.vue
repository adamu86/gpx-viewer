<template>
  <div>
    <div class="py-2 px-2 sticky! top-0! h-14 flex justify-between">
      <div class="h-12!">
        <img src="@/assets/favicon.png" alt="Logo" class="h-11! my-auto!">
      </div>
      <div class="flex gap-2 items-center">
        <MultiSelect 
          v-model="selectedFiles" 
          :options="availableFiles" 
          optionLabel="name" 
          placeholder="Select files..." 

          class="w-96" 
        />
        <FileUpload 
          ref="fileupload"
          mode="basic"
          name="demo[]"
          accept=".gpx"
          multiple
          :maxFileSize="1000000"
          :auto="true"
          @select="onFileUploadSelect"
          chooseLabel=GPX
        />
      </div>
    </div>
    <div class="mb-2 mx-2 rounded-md" id="map" style="height: calc(100vh - 64px);"></div>
  </div>
</template>

<script lang="ts" setup>
import { computed, onMounted, ref, watch } from 'vue';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import FileUpload from 'primevue/fileupload';
import MultiSelect from 'primevue/multiselect';

const files = ref<{name: string, points: { lat: number, lon: number }[]}[]>([]);
const availableFiles = ref<{name: string}[]>([]);
const selectedFiles = ref<{name: string}[]>([]);

function parseGpxToPoints(gpx: string) {
  const parser = new DOMParser();
  const xml = parser.parseFromString(gpx, 'application/xml');
  const trkpts = xml.getElementsByTagName('trkpt');
  const points: { lat: number, lon: number }[] = [];

  for (let i = 0; i < trkpts.length; i++) {
    const lat = parseFloat(trkpts[i].getAttribute('lat') || '0');
    const lon = parseFloat(trkpts[i].getAttribute('lon') || '0');

    points.push({ lat, lon });
  }

  return points;
}

let map: L.Map;

function updateMap() {
  map.eachLayer(layer => {
    if (layer instanceof L.Polyline) map.removeLayer(layer);
  });

  selectedFiles.value.forEach(file => {
    const fileData = files.value.find(f => f.name === file.name);
    if (fileData) {
      const route = L.polyline(fileData.points, { color: 'black', weight: 3, opacity: 0.5 }).addTo(map);
    }
  });
}

function onFileUploadSelect(event: { files: File[] }) {
  files.value = [];
  selectedFiles.value = [];
  availableFiles.value = [];

  event.files.forEach(file => {
    const reader = new FileReader();
    reader.onload = function(e) {
      const gpxText = e.target?.result as string;
      const points = parseGpxToPoints(gpxText);
      files.value.push({name: file.name, points: points});
      selectedFiles.value.push({name: file.name});
      availableFiles.value.push({name: file.name});
      const route = L.polyline(points, { color: 'black', weight: 3, opacity: 0.5 }).addTo(map);
      map.fitBounds(route.getBounds());
    };
    reader.readAsText(file);
  });
  
  updateMap();
}

watch(selectedFiles, () => {
  updateMap();
});

onMounted(() => {
  map = L.map('map').setView([53.23879, 23.266865], 13);

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap contributors'
  }).addTo(map);
});
</script>