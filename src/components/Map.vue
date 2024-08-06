<!-- src/components/Map.vue -->
<template>
    <div class="container">
      <div class="row justify-content-center mt-4">
        <div class="col-md-6 text-center">
          <h2 class="mb-4">Find Your Location</h2>
          <button class="btn btn-primary mb-3" @click="fetchLocation">Get My Location</button>
          <button v-if="retryButtonVisible" class="btn btn-secondary mb-3" @click="retryFetchLocation" ref="retryButton">Retry</button>
          <div id="map" class="my-4"></div>
          <div id="locationDisplay" class="alert alert-info mt-2" role="alert">
            <div v-html="formattedLocationDisplay"></div>
          </div>
        </div>
      </div>
    </div>
  </template>
  
  <script>
  import L from 'leaflet';
  import 'leaflet/dist/leaflet.css';
  
  export default {
    name: 'mapVue',
    data() {
      return {
        map: null,
        circle: null,
        accuracy: null,
        locationDisplay: '',
        retryButtonVisible: false, // Track retry button visibility
        formattedLocationDisplay: '' // Holds HTML for location display
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
  
        this.map.setView([latitude, longitude], 13);
  
        if (this.circle) {
          this.circle.setLatLng([latitude, longitude]);
          this.circle.setRadius(accuracy > 500 ? 500 : accuracy);
        } else {
          this.circle = L.circle([latitude, longitude], {
            color: 'blue',
            fillColor: '#30f',
            fillOpacity: 0.2,
            radius: accuracy > 500 ? 500 : accuracy
          }).addTo(this.map);
        }
  
        const accuracyLevel = accuracy <= 100 ? "Strong" : (accuracy <= 500 ? "Moderate" : "Low");
        this.formattedLocationDisplay = `
          <div class="location-info">
            <div><strong>Location:</strong> ${latitude.toFixed(5)}, ${longitude.toFixed(5)}</div>
            <div><strong>Accuracy:</strong> ${accuracy.toFixed(2)} meters (${accuracyLevel})</div>
            <div><strong>Displayed Radius:</strong> ${accuracy.toFixed(2)} meters</div>
          </div>`;
  
        // Update retryButtonVisible based on accuracy
        this.retryButtonVisible = accuracy > 500;
      },
      showError(error) {
        let errorMessage = "An unknown error occurred.";
  
        switch(error.code) {
          case error.PERMISSION_DENIED:
            errorMessage = "User denied the request for Geolocation.";
            break;
          case error.POSITION_UNAVAILABLE:
            errorMessage = "Location information is unavailable.";
            break;
          case error.TIMEOUT:
            errorMessage = "The request to get user location timed out.";
            break;
          case error.UNKNOWN_ERROR:
            errorMessage = "An unknown error occurred.";
            break;
        }
  
        this.locationDisplay = errorMessage;
        console.error('Geolocation error:', errorMessage);
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
  }
  
  .location-info div {
    margin-bottom: 0.5rem; /* Space between lines */
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
  </style>
  
  
  
  
  
  
  
  
  
  