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
        <v-col cols="12" lg="3" md="3" class="pa-2 d-none d-md-block">
          <div class="sticky-sidebar">
            <SideNavigation />
          </div>
        </v-col>

        <!-- Main Content -->
        <v-col cols="12" lg="6" md="6" class="pa-2 main-content-col">
          <div class="main-content-wrapper">
            <slot name="content"></slot>
          </div>
        </v-col>

        <!-- Side News -->
        <v-col cols="12" lg="3" md="3" class="pa-2 d-none d-md-block">
          <div class="sticky-sidebar">
            <SideNews />
          </div>
        </v-col>
      </v-row>
    </v-container>
    
    <!-- Mobile Bottom Navigation (outside container so it's always visible) -->
    <SideNavigation :mobile-only="true" />
  </v-layout>
</template>

<style scoped>
.dashboard-bg {
  background-color: #ebedf0;
  min-height: 100vh;
  position: relative;
  z-index: 0;
}

/* Remove extra padding on desktop */
@media (min-width: 960px) {
  .main-content-wrapper {
    padding-bottom: 0;
  }
}

.sticky-sidebar {
  position: sticky;
  top: 20px;
  max-height: calc(100vh - 40px);
  overflow-y: auto;
  z-index: 1;
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
