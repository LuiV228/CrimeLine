<template>
  <div class="container">
    <div class="row justify-content-center mt-4">
      <div class="col-md-6 text-center">
        <h2 class="mb-4">Crime Alert</h2>
        <button v-if="!formVisible" class="btn btn-primary mb-3" @click="showForm">Report</button>
        <div v-if="formVisible" class="form-container">
          <h4>Report Your Experience</h4>
          <form @submit.prevent="submitForm">
            <div class="form-group">
              <label for="crimeType">Type of Crime:</label>
              <select v-model="crimeType" id="crimeType" class="form-control" required>
                <option value="" disabled>Select a crime</option>
                <option value="experienced">Experienced a crime</option>
                <option value="witnessed">Witnessed a crime</option>
              </select>
            </div>
            <div class="form-group">
              <label for="specificCrime">Specific Crime:</label>
              <select v-model="specificCrime" id="specificCrime" class="form-control" required>
                <option value="" disabled>Select specific crime</option>
                <option value="theft">Theft</option>
                <option value="assault">Assault</option>
                <option value="vandalism">Vandalism</option>
                <!-- Add more specific crimes as needed -->
              </select>
            </div>
            <button type="submit" class="btn btn-primary">Submit</button>
            <button type="button" class="btn btn-secondary ml-2" @click="cancelForm">Cancel</button>
          </form>
        </div>
        <button v-if="retryButtonVisible" class="btn btn-secondary mb-3" @click="retryFetchLocation">Retry</button>
        <button v-if="currentLocation && !settingDangerZone && !dangerZoneFormVisible" class="btn btn-danger mb-3" @click="showDangerZoneForm">Set Danger Zone</button>
        <div v-if="dangerZoneFormVisible" class="form-container">
          <h4>Confirm Danger Zone</h4>
          <form @submit.prevent="confirmDangerZone">
            <div class="form-group">
              <label for="dangerZoneType">Type of Danger Zone:</label>
              <select v-model="dangerZoneType" id="dangerZoneType" class="form-control" required>
                <option value="" disabled>Select a type</option>
                <option value="high">High Risk</option>
                <option value="medium">Medium Risk</option>
                <option value="low">Low Risk</option>
              </select>
            </div>
            <button type="submit" class="btn btn-primary">Confirm</button>
            <button type="button" class="btn btn-secondary ml-2" @click="cancelDangerZoneSetting">Cancel</button>
          </form>
        </div>
        <div v-if="settingDangerZone" class="alert alert-warning mt-2" role="alert">
          Click on the map to set a danger zone within 1000 meters of your location.
        </div>
        <div id="map" class="my-4"></div>
        <div v-if="locationDisplay" class="alert alert-info mt-2" role="alert">
          <div v-html="locationDisplay"></div>
        </div>
        <div v-if="dangerZones.length > 0" class="danger-zones mt-4">
          <h4>Danger Zones</h4>
          <ul class="list-group">
            <li v-for="(zone, index) in dangerZones" :key="index" class="list-group-item">
              Zone {{ index + 1 }} ({{ zone.latLng.lat.toFixed(5) }}, {{ zone.latLng.lng.toFixed(5) }}, {{ zone.type }})
              <button class="btn btn-sm btn-danger float-right" @click="removeDangerZone(index)">Remove</button>
            </li>
          </ul>
        </div>
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
      settingDangerZone: false,
      formVisible: false,
      dangerZoneFormVisible: false,
      crimeType: '',
      specificCrime: '', // New data property for specific crime type
      dangerZoneType: ''
    };
  },
  mounted() {
    this.initMap();
  },
  methods: {
    initMap() {
      this.map = L.map('map').setView([51.505, -0.09], 13);

      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        maxZoom: 19,
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
      }).addTo(this.map);

      this.map.on('click', this.handleMapClick);
    },
    showForm() {
      this.formVisible = true;
    },
    cancelForm() {
      this.formVisible = false;
    },
    submitForm() {
      console.log('Crime Type:', this.crimeType);
      console.log('Specific Crime:', this.specificCrime);
      if (this.crimeType && this.specificCrime) {
        this.formVisible = false;
        this.fetchLocation();
      } else {
        console.log('Please select both the type and specific crime.');
      }
    },
    fetchLocation() {
      console.log('Fetching location...');
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(
          this.showPosition,
          this.showError,
          {
            enableHighAccuracy: true,
            timeout: 10000,
            maximumAge: 0
          }
        );
      } else {
        this.locationDisplay = "Geolocation is not supported by this browser.";
      }
    },
    retryFetchLocation() {
      console.log('Retry button clicked');
      this.fetchLocation();
    },
    showPosition(position) {
      const latitude = position.coords.latitude;
      const longitude = position.coords.longitude;
      const accuracy = position.coords.accuracy;

      console.log('Current Location:', latitude, longitude);
      console.log('Accuracy:', accuracy);

      this.currentLocation = [latitude, longitude];
      this.map.setView(this.currentLocation, 13);

      if (this.circle) {
        this.circle.setLatLng(this.currentLocation);
        this.circle.setRadius(accuracy > 500 ? 500 : accuracy);
      } else {
        this.circle = L.circle(this.currentLocation, {
          color: 'blue',
          fillColor: '#30f',
          fillOpacity: 0.2,
          radius: accuracy > 500 ? 500 : accuracy
        }).addTo(this.map);
      }

      const accuracyLevel = accuracy <= 100 ? "Strong" : (accuracy <= 500 ? "Moderate" : "Low");
      this.locationDisplay = `
        <div class="location-info">
          <div class="location-item"><strong>Coordinates:</strong> Latitude: ${latitude.toFixed(5)}, Longitude: ${longitude.toFixed(5)}</div>
          <div class="location-item"><strong>Accuracy:</strong> We are confident that your location is within ${accuracy.toFixed(2)} meters (${accuracyLevel})</div>
          <div class="location-item"><strong>Displayed Radius:</strong> The circle on the map shows an area of ${accuracy.toFixed(2)} meters around your location</div>
        </div>`;

      this.retryButtonVisible = accuracy > 500;
    },
    showError(error) {
      let errorMessage = "An unknown error occurred.";

      switch (error.code) {
        case error.PERMISSION_DENIED:
          errorMessage = "You denied the request for your location. Please enable location services in your browser.";
          break;
        case error.POSITION_UNAVAILABLE:
          errorMessage = "Your location is currently unavailable.";
          break;
        case error.TIMEOUT:
          errorMessage = "The request to get your location timed out. Please try again.";
          break;
        case error.UNKNOWN_ERROR:
          errorMessage = "An unknown error occurred while fetching your location.";
          break;
      }

      this.locationDisplay = errorMessage;
      console.error('Geolocation error:', errorMessage);
    },
    showDangerZoneForm() {
      this.dangerZoneFormVisible = true;
    },
    cancelDangerZoneSetting() {
      this.settingDangerZone = false;
      this.dangerZoneFormVisible = false;
    },
    confirmDangerZone() {
      console.log('Danger Zone Type:', this.dangerZoneType);
      if (this.dangerZoneType) {
        this.dangerZoneFormVisible = false;
        this.settingDangerZone = true;
      } else {
        console.log('Please select a type for the danger zone.');
      }
    },
    handleMapClick(e) {
      if (this.settingDangerZone) {
        const selectedPoint = e.latlng;
        const distance = this.map.distance(this.currentLocation, selectedPoint);

        if (distance <= 500) {
          const dangerZone = {
            latLng: selectedPoint,
            circle: L.circle(selectedPoint, {
              color: 'red',
              fillColor: '#f03',
              fillOpacity: 0.5,
              radius: 100
            }).addTo(this.map),
            type: this.dangerZoneType // Store the type of the danger zone
          };

          this.dangerZones.push(dangerZone);
          this.settingDangerZone = false;
          this.dangerZoneFormVisible = false;
        } else {
          this.locationDisplay = `
            <div class="alert alert-danger mt-2" role="alert">
              The selected point is more than 500 meters away from your location. Please select a point closer to your current location.
            </div>`;
        }
      }
    },
    removeDangerZone(index) {
      this.map.removeLayer(this.dangerZones[index].circle);
      this.dangerZones.splice(index, 1);
    }
  }
}
</script>

<style scoped>
#map {
  height: 400px;
  width: 100%;
}

#locationDisplay {
  white-space: pre-line;
}

.location-info {
  text-align: left;
  font-size: 1rem;
}

.location-item {
  margin-bottom: 0.5rem;
  font-size: 1.1rem;
}

.container {
  max-width: 800px;
}

h2 {
  color: #333;
}

.btn-primary {
  background-color: #007bff;
  border-color: #007bff;
}

.btn-secondary {
  background-color: #6c757d;
  border-color: #6c757d;
}

.btn-danger {
  background-color: #dc3545;
  border-color: #dc3545;
}

.btn-warning {
  background-color: #ffc107;
  border-color: #ffc107;
}

.danger-zones {
  margin-top: 20px;
}

.danger-zones ul {
  list-style-type: none;
  padding: 0;
}

.danger-zones li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  border: 1px solid #ccc;
  margin-bottom: 5px;
  border-radius: 4px;
}

.alert {
  font-size: 1rem;
}

.form-container {
  margin-top: 20px;
  padding: 20px;
  border: 1px solid #ccc;
  border-radius: 4px;
  background-color: #f9f9f9;
}

.form-group {
  margin-bottom: 15px;
}

.form-control {
  width: 100%;
}

.ml-2 {
  margin-left: 10px;
}
</style>

