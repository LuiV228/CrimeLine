<!-- src/components/LoginForm.vue -->
<template>
    <div class="login-container">
      <form @submit.prevent="handleLogin" class="login-form">
        <h2>Login</h2>
        <div class="form-group">
          <label for="username">Username</label>
          <input v-model="username" type="text" id="username" required />
        </div>
        <div class="form-group">
          <label for="password">Password</label>
          <input v-model="password" type="password" id="password" required />
        </div>
        <button type="submit" class="btn btn-primary">Login</button>
        <div v-if="errorMessage" class="alert alert-danger mt-3">{{ errorMessage }}</div>
      </form>
    </div>
  </template>
  
  <script>
  import axios from 'axios';
  
  export default {
    data() {
      return {
        username: '',
        password: '',
        errorMessage: ''
      };
    },
    methods: {
      async handleLogin() {
        try {
          const response = await axios.post('http://localhost:3000/api/login', {
            username: this.username,
            password: this.password
          });
  
          if (response.data.success) {
            this.$router.push('/dashboard'); // Redirect to a dashboard or home page on successful login
          } else {
            this.errorMessage = response.data.message;
          }
        } catch (error) {
          this.errorMessage = 'An error occurred during login.';
          console.error('Login error:', error);
        }
      }
    }
  }
  </script>
  
  <style scoped>
  .login-container {
    max-width: 400px;
    margin: auto;
    padding: 2rem;
    border: 1px solid #ccc;
    border-radius: 5px;
  }
  
  .login-form {
    display: flex;
    flex-direction: column;
  }
  
  .form-group {
    margin-bottom: 1rem;
  }
  
  label {
    display: block;
    margin-bottom: 0.5rem;
  }
  
  input {
    width: 100%;
    padding: 0.5rem;
    border: 1px solid #ddd;
    border-radius: 3px;
  }
  
  button {
    padding: 0.5rem;
    border: none;
    border-radius: 3px;
    background-color: #007bff;
    color: white;
    cursor: pointer;
  }
  
  button:hover {
    background-color: #0056b3;
  }
  
  .alert {
    padding: 0.5rem;
    border-radius: 3px;
  }
  </style>
  