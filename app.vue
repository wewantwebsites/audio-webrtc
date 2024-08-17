<script setup>
import { useRoute } from 'vue-router';

const route = useRoute();

const links = [
  {
    label: 'Home',
    icon: 'i-heroicons-home',
    to: '/',
  },
  {
    label: 'About',
    icon: 'i-heroicons-information-circle',
    to: '/about',
  },
];
</script>

<template>
  <div>
    <NuxtRouteAnnouncer />
    <UHorizontalNavigation
      :links="links"
      class="border-b border-gray-200 dark:border-gray-800"
    />
    <section class="p-4">
      <NuxtPage />
    </section>
  </div>
</template>
