<template>
    <div class="dashboard-container">
      <h2>Welcome to Crime Alert Dashboard</h2>
      <button @click="manageAlerts" class="btn btn-primary">Manage Alerts</button>
      <div id="map" style="height: 400px; width: 100%;"></div>
    </div>
  </template>
  
  <script>
  import axios from 'axios';
  import L from 'leaflet';
  import 'leaflet/dist/leaflet.css';
  
  export default {
    name: 'DashboardVue',
    data() {
      return {
        map: null,
        alerts: []
      };
    },
    mounted() {
      this.initMap();
      this.fetchAlerts();
    },
    methods: {
      initMap() {
        this.map = L.map('map').setView([51.505, -0.09], 13);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
          maxZoom: 19
        }).addTo(this.map);
      },
      async fetchAlerts() {
        try {
          const userId = 1; // Replace with actual user ID
          const response = await axios.get(`http://localhost:3000/api/alerts/${userId}`);
          this.alerts = response.data.alerts;
  
          this.alerts.forEach(alert => {
            L.marker([alert.location.latitude, alert.location.longitude])
              .addTo(this.map)
              .bindPopup(`<b>${alert.title}</b><br>${alert.description}`);
          });
        } catch (error) {
          console.error('Error fetching alerts:', error);
        }
      },
      manageAlerts() {
        this.$router.push('/manage-alerts');
      }
    }
  }
  </script>
  
  <style scoped>
  .dashboard-container {
    max-width: 800px;
    margin: 0 auto;
    padding: 20px;
    text-align: center;
  }
  
  h2 {
    margin-bottom: 20px;
  }
  
  .btn-primary {
    background-color: #007bff;
    border-color: #007bff;
  }
  </style>
  
  