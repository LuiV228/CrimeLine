<template>
  <div class="container">
    <div class="content">
      <h2 class="title">Crime Alert</h2>
      <button v-if="!formVisible && !modalVisible" class="btn btn-primary" @click="showModal">Report</button>

      <!-- Modal -->
      <div v-if="modalVisible" class="modal">
        <div class="modal-content">
          <h4 v-if="currentQuestion">{{ currentQuestion.title }}</h4>
          <div class="modal-options" v-if="currentQuestion">
            <button 
              v-for="(option, index) in currentQuestion.options" 
              :key="index" 
              :class="['btn', { 'selected': selectedOption === option.value }]" 
              @click="selectAnswer(option.value)">
              {{ option.label }}
            </button>
          </div>
          <div class="modal-navigation">
            <button 
              class="btn btn-secondary" 
              @click="previousQuestion"
              :disabled="questionIndex === 0">
              Previous
            </button>
            <button 
              v-if="!isLastQuestion" 
              class="btn btn-primary" 
              @click="nextQuestion"
              :disabled="!selectedOption">
              Next
            </button>
            <button 
              v-if="isLastQuestion" 
              class="btn btn-primary" 
              @click="getLocation">
              Get Location
            </button>
          </div>
        </div>
      </div>

      <div v-if="retryButtonVisible" class="ui-message">
        <p>Failed to fetch accurate location. 
          <button class="btn btn-secondary" @click="retryFetchLocation">Retry</button>
        </p>
      </div>

      <!-- Render map only if the location is available -->
      <div v-if="currentLocation && !riskLevelFormVisible && !dangerZoneFormVisible && !mapVisible" class="map-setup">
        <button class="btn btn-danger" @click="enableRiskLevelForm">Set Danger Zone</button>
      </div>

      <div v-if="mapVisible" id="map" class="map"></div> <!-- Map container -->

      <div v-if="dangerZones.length > 0" class="danger-zones">
        <h4 class="danger-zones-title">Danger Zones</h4>
        <ul>
          <li v-for="(zone, index) in dangerZones" :key="index" class="danger-zone-item">
            Zone {{ index + 1 }} ({{ zone.latLng.lat.toFixed(5) }}, {{ zone.latLng.lng.toFixed(5) }}) - {{ zone.riskLevel }}
            <button class="btn btn-sm btn-danger" @click="removeDangerZone(index)">Remove</button>
          </li>
        </ul>
      </div>

      <div v-if="locationDisplay" class="ui-message">
        <div v-html="locationDisplay"></div>
      </div>
    </div>
  </div>
</template>

<script>
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';

export default {
  name: 'MapVue',
  data() {
    return {
      map: null,
      circle: null,
      locationDisplay: '',
      retryButtonVisible: false,
      dangerZones: [],
      currentLocation: null,
      modalVisible: false,
      mapVisible: false,
      riskLevelFormVisible: false,
      dangerZoneFormVisible: false,
      crimeType: '',
      specificCrime: '',
      riskLevel: '',
      dangerZoneRadius: 700,
      isSettingDangerZone: false,
      areaName: '',
      questionIndex: 0,
      selectedOption: null,
      questions: [
        {
          title: 'What type of crime did you encounter?',
          options: [
            { label: 'Experienced a crime', value: 'experienced' },
            { label: 'Witnessed a crime', value: 'witnessed' },
            { label: 'Heard about a crime', value: 'heard' },
          ]
        },
        {
          title: 'What specific crime did you encounter?',
          options: [
            { label: 'Theft', value: 'theft' },
            { label: 'Assault', value: 'assault' },
            { label: 'Vandalism', value: 'vandalism' },
            { label: 'Burglary', value: 'burglary' },
            { label: 'Robbery', value: 'robbery' },
            { label: 'Fraud', value: 'fraud' },
            { label: 'Harassment', value: 'harassment' },
            { label: 'Other', value: 'other' },
          ]
        },
        {
          title: 'Where did the crime occur?',
          options: [
            { label: 'On the street', value: 'street' },
            { label: 'In a building (residential)', value: 'residential' },
            { label: 'In a building (commercial)', value: 'commercial' },
            { label: 'At a public event', value: 'event' },
            { label: 'In a vehicle', value: 'vehicle' },
            { label: 'In a park or open space', value: 'public place' },
            { label: 'Unknown', value: 'unknown' },
          ]
        },
        {
          title: 'When did the crime occur?',
          options: [
            { label: 'Just now', value: 'just now' },
            { label: 'Within the last hour', value: 'last hour' },
            { label: 'Today', value: 'today' },
            { label: 'Yesterday', value: 'yesterday' },
            { label: 'Last week', value: 'last week' },
          ]
        },
        {
          title: 'How many suspects were involved?',
          options: [
            { label: '1', value: '1 suspect' },
            { label: '2-3', value: '2-3 suspects' },
            { label: '4-5', value: '4-5 suspects' },
            { label: 'More than 5', value: 'more than 5 suspects' },
            { label: 'Unknown', value: 'unknown suspect(s)' },
          ]
        },
        {
          title: 'What was the suspect(s) wearing?',
          options: [
            { label: 'Dark clothing', value: 'dark clothing' },
            { label: 'Light clothing', value: 'light clothing' },
            { label: 'Casual wear', value: 'casual wear' },
            { label: 'Formal wear', value: 'formal wear' },
            { label: 'Hooded jacket or hoodie', value: 'hooded jacket' },
            { label: 'Mask or face covering', value: 'mask' },
            { label: 'Unknown', value: 'unknown' },
          ]
        },
        {
          title: 'Were any weapons involved?',
          options: [
            { label: 'Yes, a firearm', value: 'firearm' },
            { label: 'Yes, a knife or sharp object', value: 'knife' },
            { label: 'Yes, another type of weapon', value: 'other' },
            { label: 'No weapons', value: 'unarmed' },
            { label: 'Unknown', value: 'unknown' },
          ]
        },
        {
          title: 'Was a vehicle involved?',
          options: [
            { label: 'Yes, a car', value: 'car' },
            { label: 'Yes, a motorcycle', value: 'motorcycle/scooter' },
            { label: 'Yes, a bicycle', value: 'bicycle' },
            { label: 'No vehicle', value: 'no vehicle' },
            { label: 'Unknown', value: 'unknown' },
          ]
        },
        {
          title: 'Were there any injuries?',
          options: [
            { label: 'Yes, minor injuries', value: 'minor injuries' },
            { label: 'Yes, serious injuries', value: 'serious injuries' },
            { label: 'Yes, fatal injuries', value: 'fatal injuries' },
            { label: 'No injuries', value: 'no injuries' },
            { label: 'Unknown', value: 'unknown' },
          ]
        },
        {
          title: 'Was anything stolen?',
          options: [
            { label: 'Yes, personal belongings', value: 'personal belongings' },
            { label: 'Yes, cash or valuables', value: 'cash or valuables' },
            { label: 'Yes, electronics', value: 'electronics' },
            { label: 'Yes, a vehicle', value: 'vehicle' },
            { label: 'No, nothing was stolen', value: 'no' },
            { label: 'Unknown', value: 'unknown' },
          ]
        },
        {
          title: 'In which direction did the suspect(s) flee?',
          options: [
            { label: 'North', value: 'north' },
            { label: 'South', value: 'south' },
            { label: 'East', value: 'east' },
            { label: 'West', value: 'west' },
            { label: 'Unknown', value: 'unknown' },
          ]
        },
        {
          title: 'Have the authorities been contacted?',
          options: [
            { label: 'Yes, they have been contacted', value: 'yes' },
            { label: 'No, not yet', value: 'no' },
          ]
        },
      ]
    };
  },
  computed: {
    currentQuestion() {
      return this.questions[this.questionIndex];
    },
    isLastQuestion() {
      return this.questionIndex === this.questions.length - 1;
    }
  },
  methods: {
    showModal() {
      this.modalVisible = true;
    },
    selectAnswer(value) {
      this.selectedOption = value;
    },
    nextQuestion() {
      if (this.questionIndex < this.questions.length - 1) {
        this.questionIndex++;
        this.selectedOption = null; // Clear selection for the next question
      }
    },
    previousQuestion() {
      if (this.questionIndex > 0) {
        this.questionIndex--;
        this.selectedOption = null; // Clear selection for the previous question
      }
    },
    getLocation() {
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(
          (position) => {
            const lat = position.coords.latitude;
            const lng = position.coords.longitude;
            this.currentLocation = { lat, lng };
            this.locationDisplay = `<strong>Location:</strong> (${lat.toFixed(5)}, ${lng.toFixed(5)})`;
            this.mapVisible = true;

            // Initialize the map
            this.$nextTick(() => {
              this.initMap();
            });
          },
          () => {
            alert('Failed to retrieve location. Please enable location services and try again.');
          },
          { enableHighAccuracy: true }
        );
      } else {
        alert('Geolocation is not supported by this browser.');
      }

      this.modalVisible = false; // Close the modal after getting location
    },
    enableRiskLevelForm() {
      this.riskLevelFormVisible = true;
      this.isSettingDangerZone = true;
    },
    retryFetchLocation() {
      this.retryButtonVisible = false;
      this.getLocation();
    },
    initMap() {
      this.map = L.map('map').setView([this.currentLocation.lat, this.currentLocation.lng], 13);

      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        maxZoom: 19,
      }).addTo(this.map);

      L.marker([this.currentLocation.lat, this.currentLocation.lng])
        .addTo(this.map)
        .bindPopup('You are here')
        .openPopup();

      this.map.on('click', this.handleMapClick);
    },
    handleMapClick(e) {
      if (this.isSettingDangerZone) {
        const latLng = e.latlng;
        const zone = {
          latLng,
          riskLevel: this.riskLevel,
        };

        this.dangerZones.push(zone);
        this.drawDangerZone(latLng, this.dangerZoneRadius);
        this.isSettingDangerZone = false; // Reset the flag
      }
    },
    drawDangerZone(latLng, radius) {
      if (this.circle) {
        this.map.removeLayer(this.circle);
      }
      this.circle = L.circle(latLng, {
        color: 'red',
        fillColor: '#f03',
        fillOpacity: 0.5,
        radius: radius,
      }).addTo(this.map);
    },
    removeDangerZone(index) {
      this.dangerZones.splice(index, 1);
      if (this.circle) {
        this.map.removeLayer(this.circle);
      }
    }
  },
  mounted() {
    this.fetchLocation();
  }
};
</script>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  text-align: center;
}
.content {
  margin-top: 20px;
}
.title {
  font-size: 24px;
  margin-bottom: 20px;
}
.btn {
  margin: 10px;
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}
.btn-primary {
  background-color: #007bff;
  color: white;
}
.btn-secondary {
  background-color: #6c757d;
  color: white;
}
.btn-danger {
  background-color: #dc3545;
  color: white;
}
.selected {
  border: 2px solid #007bff;
}
.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}
.modal-content {
  background: white;
  padding: 20px;
  border-radius: 5px;
  text-align: left;
  width: 300px;
}
.map-setup {
  margin-top: 20px;
}
.map {
  height: 400px;
  width: 100%;
}
.danger-zones {
  margin-top: 20px;
}
.danger-zone-item {
  margin: 10px 0;
  padding: 10px;
  background-color: #f8f9fa;
  border-radius: 5px;
  text-align: left;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.danger-zones-title {
  font-size: 18px;
  margin-bottom: 10px;
}
.ui-message {
  margin-top: 20px;
  padding: 15px;
  background-color: #f8f9fa;
  border-radius: 5px;
  text-align: left;
}
</style>
