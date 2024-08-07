<!-- src/components/Map.vue -->
<template>
  <div class="container">
    <div class="row justify-content-center mt-4">
      <div class="col-md-6 text-center">
        <h2 class="mb-4">Crime Alert</h2>
        <button class="btn btn-primary mb-3" @click="fetchLocation">Get My Location</button>
        <button v-if="retryButtonVisible" class="btn btn-secondary mb-3" @click="retryFetchLocation" ref="retryButton">Retry</button>
        <button v-if="currentLocation" class="btn btn-danger mb-3" @click="enableDangerZoneSetting">Set Danger Zone</button>
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
              Zone {{ index + 1 }} ({{ zone.latLng.lat.toFixed(5) }}, {{ zone.latLng.lng.toFixed(5) }})
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
      retryButtonVisible: false, // Track retry button visibility
      dangerZones: [], // Track danger zones
      currentLocation: null, // Store the current location
      settingDangerZone: false, // Track if we're setting a danger zone
      panicMarker: null, // Track the panic marker
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

      this.map.on('click', this.handleMapClick); // Attach click event handler
    },
    fetchLocation() {
      console.log('Fetching location...'); // Debugging line
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(this.showPosition, this.showError, { 
          enableHighAccuracy: true, 
          timeout: 10000, 
          maximumAge: 0 
        });
      } else {
        this.locationDisplay = "Geolocation is not supported by this browser.";
      }
    },
    retryFetchLocation() {
      console.log('Retry button clicked'); // Debugging line
      this.fetchLocation(); // Trigger the fetchLocation method
    },
    showPosition(position) {
      const latitude = position.coords.latitude;
      const longitude = position.coords.longitude;
      const accuracy = position.coords.accuracy;

      console.log('Current Location:', latitude, longitude); // Debugging line
      console.log('Accuracy:', accuracy); // Debugging line

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

      // Update retryButtonVisible based on accuracy
      this.retryButtonVisible = accuracy > 500;
    },
    showError(error) {
      let errorMessage = "An unknown error occurred.";

      switch(error.code) {
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
    enableDangerZoneSetting() {
      this.settingDangerZone = true;
    },
    handleMapClick(event) {
      if (this.settingDangerZone) {
        const clickLatLng = event.latlng;
        const distance = this.map.distance(this.currentLocation, clickLatLng);
        
        if (distance <= 500) {
          const dangerZone = L.circle(clickLatLng, {
            color: 'orange',
            fillColor: '#f60',
            fillOpacity: 0.5,
            radius: 100
          }).addTo(this.map);

          this.dangerZones.push({ circle: dangerZone, latLng: clickLatLng });
          this.settingDangerZone = false; // Disable setting danger zone mode
        } else {
          this.locationDisplay = `
            <div class="alert alert-danger mt-2" role="alert">
              The selected point is more than 1000 meters away from your location. Please select a point closer to your current location.
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
  white-space: pre-line; /* Preserve line breaks in the alert */
}

.location-info {
  text-align: left;
  font-size: 1rem;
}

.location-item {
  margin-bottom: 0.5rem; /* Space between lines */
  font-size: 1.1rem; /* Slightly increase font size */
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
  font-size: 1rem; /* Increase font size for better readability */
}
</style>







  
  
  
  
  
  
  
  
  
  