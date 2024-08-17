<template>
  <div>
    <h1>Details for contact without a ID</h1>

    <UContainer class="flex">
      <UInput
        v-model="id"
        type="number"
        step="1"
        min="0"
        label="Contact Name"
        placeholder="Enter contact name"
        :ui="{ rounded: 'rounded-md rounded-r-none' }"
      />
      <UButton
        color="primary"
        class="flex justify-center"
        :disabled="!id"
        @click="() => router.push('/nested/' + id)"
        :ui="{ rounded: 'rounded-md rounded-l-none' }"
        >Go!</UButton
      >
    </UContainer>
  </div>
</template>
<script setup>
import { ref } from 'vue';
const router = useRouter();
const id = ref('');
</script>
