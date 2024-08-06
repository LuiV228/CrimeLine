<template>
    <div>
      <h2>Register</h2>
      <form @submit.prevent="register">
        <div>
          <label for="username">Username:</label>
          <input type="text" v-model="username" required>
        </div>
        <div>
          <label for="password">Password:</label>
          <input type="password" v-model="password" required>
        </div>
        <button type="submit">Register</button>
      </form>
    </div>
  </template>
  
  <script>
  import axios from 'axios';
  
  export default {
    data() {
      return {
        username: '',
        password: ''
      };
    },
    methods: {
      async register() {
        try {
          const response = await axios.post('http://localhost:3000/api/users/register', {
            username: this.username,
            password: this.password
          });
          alert(response.data.message);
          // Redirect to login page or perform other actions on successful registration
        } catch (error) {
          alert(error.response.data.message);
        }
      }
    }
  };
  </script>
  