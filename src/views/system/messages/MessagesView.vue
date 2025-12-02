<script setup>
import { ref } from 'vue'
import DashboardLayout from '@/components/layout/DashboardLayout.vue'
import MessagesLayout from '@/components/layout/messages/MessagesLayout.vue'
</script>

<template>
  <DashboardLayout>
    <template #content>
      <MessagesLayout />
    </template>
  </DashboardLayout>
</template>
