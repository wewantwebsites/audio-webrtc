<template>
  <UContainer>
    <section class="grid grid-cols-1 md:grid-cols-3">
      <UCard>
        <template #header>
          <h1>Add Contact</h1>
        </template>

        <section>
          <p>
            Lorem ipsum dolor sit amet consectetur adipisicing elit. Corrupti
            velit, praesentium totam possimus quos voluptate atque animi
            inventore cupiditate molestiae facere, commodi rem enim fugiat
            consequatur, a est numquam magni.
          </p>
        </section>
        <template #footer>
          <UButton
            color="primary"
            class="w-full flex justify-center"
            @click="() => console.log('contact added!')"
            >Add Contact</UButton
          >
        </template>
      </UCard>
      <UDivider orientation="vertical" label="OR" />
      <UCard>
        <template #header>
          <h1>Make a Call</h1>
        </template>

        <section>
          <p>
            Lorem ipsum dolor sit amet consectetur adipisicing elit. Corrupti
            velit, praesentium totam possimus quos voluptate atque animi
            inventore cupiditate molestiae facere, commodi rem enim fugiat
            consequatur, a est numquam magni.
          </p>
        </section>

        <template #footer>
          <UButton
            color="primary"
            class="w-full flex justify-center"
            @click="() => console.log('calling contact')"
            >Call a Contact
          </UButton>
        </template>
      </UCard>
    </section>
  </UContainer>
</template>
