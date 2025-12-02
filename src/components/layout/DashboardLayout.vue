<script>
import SideNavigation from '@/components/layout/SideNavigation.vue'
import SideNews from '@/components/layout/SideNews.vue'

export default {
  components: {
    SideNavigation,
    SideNews
  }
}
</script>

<template>
  <v-layout class="dashboard-bg">
    <v-container fluid class="pa-4">
      <v-row class="fill-height">
        <!-- Side Navigation -->
        <v-col cols="12" lg="3" md="3" class="pa-2">
          <div class="sticky-sidebar">
            <SideNavigation />
          </div>
        </v-col>

        <!-- Main Content -->
        <v-col cols="12" lg="6" md="6" class="pa-2">
          <slot name="content"></slot>
        </v-col>

        <!-- Side News -->
        <v-col cols="12" lg="3" md="3" class="pa-2">
          <div class="sticky-sidebar">
            <SideNews />
          </div>
        </v-col>
      </v-row>
    </v-container>
  </v-layout>
</template>

<style scoped>
.dashboard-bg {
  background-color: #ebedf0;
  min-height: 100vh;
}

.sticky-sidebar {
  position: sticky;
  top: 20px;
  max-height: calc(100vh - 40px);
  overflow-y: auto;
}

.sticky-sidebar::-webkit-scrollbar {
  width: 6px;
}

.sticky-sidebar::-webkit-scrollbar-track {
  background: transparent;
}

.sticky-sidebar::-webkit-scrollbar-thumb {
  background: rgba(0, 0, 0, 0.2);
  border-radius: 3px;
}

.sticky-sidebar::-webkit-scrollbar-thumb:hover {
  background: rgba(0, 0, 0, 0.3);
}
</style>
