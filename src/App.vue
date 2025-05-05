<script setup>
import Header from './components/Header.vue'
import MainPage from './components/MainPage.vue'
import { ref } from 'vue'

const restaurant = ref('');

function restaurantLoaded(data) {
  restaurant.value = data;
}
</script>

<template>
  <header>
    <Header :restaurant="restaurant"/>
  </header>

  <main>
    <MainPage @restaurant-loaded="restaurantLoaded"/>
  </main>
</template>

<style scoped>

</style>
