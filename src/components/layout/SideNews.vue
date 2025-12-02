<script setup>
import { useDisplay } from 'vuetify'

// Get mobile status from Vuetify's useDisplay
const { mobile } = useDisplay()

// Props
defineProps({
  modelValue: {
    type: Boolean,
    required: true
  },
  permanent: {
    type: Boolean,
    default: false
  }
})
</script>

<template>
  <div class="side-news-card rounded-xl shadow-lg overflow-hidden h-full">
    <div class="news-content bg-white">
      <div class="news-header text-center py-4 px-4">
        <h3 class="text-green-darken-3 font-weight-bold text-h6 mb-2">
          <v-icon color="green-darken-3" size="28" class="mr-2">mdi-facebook</v-icon>
          CCIS News
        </h3>
        <p class="text-sm text-gray-600">Stay updated with CCIS</p>
      </div>

      <v-divider class="mx-4"></v-divider>

      <div class="pa-4">
        <!-- Show Facebook Embed iframe if not on mobile -->
        <div v-if="!mobile" class="rounded-lg overflow-hidden shadow-md">
          <iframe
            src="https://www.facebook.com/plugins/page.php?href=https%3A%2F%2Fwww.facebook.com%2Fcciscarsu&tabs=timeline&width=300&height=689&small_header=false&adapt_container_width=true&hide_cover=false&show_facepile=true&appId"
            width="100%"
            height="700"
            style="border: none; overflow: hidden"
            scrolling="no"
            frameborder="0"
            allowfullscreen="true"
            allow="autoplay; clipboard-write; encrypted-media; picture-in-picture; web-share"
          ></iframe>
        </div>

        <!-- Fallback message when on mobile -->
        <div v-else class="text-center">
          <div class="mb-4 rounded-lg overflow-hidden shadow-md">
            <img src="/images/ccis.png" alt="CCIS Logo" class="w-full" />
          </div>
          <p class="text-center text-gray-700 mb-4 text-sm">
            Embedding is not supported on mobile devices.
          </p>
          <v-btn
            href="https://www.facebook.com/cciscarsu"
            target="_blank"
            prepend-icon="mdi-facebook"
            color="green-darken-3"
            variant="flat"
            block
            rounded="lg"
            class="font-weight-bold"
          >
            Visit CCIS Page
          </v-btn>
        </div>
      </div>
    </div>
  </div>
</template>
