<script setup>
const handleClick = () => {
  alert('Hello from the about page!');
};
</script>

<template>
  <UCard>
    <template #header>
      <h1>About Page</h1>
    </template>

    <section>
      <p>
        Lorem ipsum dolor sit amet consectetur adipisicing elit. Corrupti velit,
        praesentium totam possimus quos voluptate atque animi inventore
        cupiditate molestiae facere, commodi rem enim fugiat consequatur, a est
        numquam magni.
      </p>
    </section>

    <template #footer>
      <UButton @click="handleClick" @keypress="handleClick">Click me</UButton>
    </template>
  </UCard>
</template>
