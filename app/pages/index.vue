<template>
    <div>
        Router Page
    </div>
  <div>
    <NuxtLink to="/overlay">Overlay</NuxtLink>
    <NuxtLink to="/admin">Admin Controller</NuxtLink>
  </div>
</template>

<script setup lang="ts">
// You can add any component-specific logic here
</script>

<style scoped>
/* Add your styles here */
div {
  font-family: Arial, sans-serif;
  text-align: center;
  margin-top: 50px;
}

h1 {
  color: #333;
}

p {
  color: #666;
}

a {
  display: inline-block;
  margin-top: 20px;
  padding: 10px 20px;
  background-color: #007bff;
  color: white;
  text-decoration: none;
  border-radius: 5px;
}

a:hover {
  background-color: #0056b3;
}
</style>
