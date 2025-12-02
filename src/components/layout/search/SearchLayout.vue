<script setup>
import { ref, computed, onMounted } from 'vue'
import { supabase } from '@/utils/supabase.js'

// State variables
const searchQuery = ref('') // User's search input
const posts = ref([]) // Data from Supabase
// URL of the image
const profileUrl = 'https://ndmbunubneumkuadlylz.supabase.co/storage/v1/object/public/images/'

const fetchPosts = async () => {
  const { data, error } = await supabase
    .rpc('get_posts_with_user_info') // Access the function
    .select('*') // Select all columns from the function table

  if (error) {
    console.error('Error fetching posts:', error)
  } else {
    posts.value = data // Store the result in posts
  }
}

// Filter posts based on search query
const filteredPosts = computed(() => {
  const query = searchQuery.value.toLowerCase() // Normalize case
  return posts.value.filter(
    (post) =>
      post.item_name.toLowerCase().includes(query) || post.description.toLowerCase().includes(query)
  )
})

// Fetch posts on component mount
onMounted(() => {
  fetchPosts()
})

// Listen for profile updates to refresh posts
window.addEventListener('profile-updated', fetchPosts)

// Format date to readable string
const formatDate = (dateString) => {
  if (!dateString) return ''
  const date = new Date(dateString)
  const now = new Date()
  const diffInSeconds = Math.floor((now - date) / 1000)
  
  if (diffInSeconds < 60) return 'Just now'
  if (diffInSeconds < 3600) return `${Math.floor(diffInSeconds / 60)} minutes ago`
  if (diffInSeconds < 86400) return `${Math.floor(diffInSeconds / 3600)} hours ago`
  if (diffInSeconds < 604800) return `${Math.floor(diffInSeconds / 86400)} days ago`
  
  return date.toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' })
}
</script>

<template>
  <v-container>
    <!-- Search Card -->
    <v-row justify="center" class="mt-4">
      <v-col cols="12">
        <v-card class="search-card rounded-xl overflow-hidden" elevation="2">
          <div class="search-header pa-5">
            <div class="d-flex align-center mb-3">
              <v-icon size="32" color="white" class="mr-3">mdi-magnify</v-icon>
              <div>
                <h2 class="text-h6 text-white font-weight-bold mb-0">Search Lost Items</h2>
                <p class="text-body-2 text-white text-opacity-90 mb-0">
                  Find items reported by the community
                </p>
              </div>
            </div>
            <v-text-field
              v-model="searchQuery"
              placeholder="Search by item name or description..."
              variant="outlined"
              density="comfortable"
              rounded="lg"
              bg-color="white"
              color="green-darken-3"
              prepend-inner-icon="mdi-magnify"
              hide-details
              single-line
              class="search-input"
            ></v-text-field>
          </div>
        </v-card>
      </v-col>
    </v-row>

    <!-- Search Results -->
    <v-row v-if="searchQuery !== ''" class="mt-2">
      <v-col cols="12">
        <p class="text-body-2 text-grey-darken-1 mb-3">
          {{ filteredPosts.length }} result{{ filteredPosts.length !== 1 ? 's' : '' }} found
        </p>
      </v-col>

      <v-col cols="12" v-for="post in filteredPosts" :key="post.post_id">
        <v-card class="post-card rounded-xl mb-4 overflow-hidden" elevation="3">
          <!-- Post Header -->
          <div class="post-header pa-4 d-flex align-center">
            <v-avatar size="56" class="mr-3 elevation-2">
              <v-img
                v-if="post.profile_pic && post.profile_pic !== ''"
                :src="
                  post.profile_pic.startsWith('http')
                    ? post.profile_pic
                    : profileUrl + post.profile_pic
                "
                alt="User Avatar"
                cover
              />
              <v-img
                v-else
                :src="post.avatar_url || 'default-avatar-url.png'"
                alt="Google profile"
                cover
              />
            </v-avatar>
            <div>
              <h3 class="text-green-darken-3 font-weight-bold text-subtitle-1 mb-0">
                {{
                  post.firstname && post.lastname
                    ? post.firstname + ' ' + post.lastname
                    : post.full_name
                }}
              </h3>
              <p class="text-caption text-grey-darken-1 mb-0">{{ formatDate(post.created_at) }}</p>
            </div>
          </div>

          <!-- Post Image -->
          <div class="post-image-container">
            <v-img
              v-if="post.image"
              :src="`https://ndmbunubneumkuadlylz.supabase.co/storage/v1/object/public/items/${post.image}`"
              :alt="post.item_name || 'Post Image'"
              contain
              class="post-image"
              max-height="500"
            />
          </div>

          <!-- Post Content -->
          <div class="pa-4">
            <h2 class="text-h6 text-green-darken-3 font-weight-bold mb-2">{{ post.item_name }}</h2>
            <p class="text-body-2 text-grey-darken-2 mb-3">{{ post.description }}</p>
          </div>

          <!-- Post Actions -->
          <v-divider></v-divider>
          <v-card-actions class="pa-3">
            <v-spacer></v-spacer>
            <v-btn
              color="orange-darken-2"
              variant="flat"
              prepend-icon="mdi-facebook-messenger"
              rounded="lg"
              size="default"
              class="font-weight-medium"
              :href="post.facebook_link"
              target="_blank"
              rel="noopener"
            >
              Send Message
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-col>
    </v-row>

    <!-- Empty State -->
    <v-row v-else class="mt-8">
      <v-col cols="12" class="text-center">
        <v-icon size="80" color="grey-lighten-1" class="mb-4">mdi-magnify</v-icon>
        <h3 class="text-h6 text-grey-darken-1 mb-2">Start searching</h3>
        <p class="text-body-2 text-grey">Enter keywords to find lost items</p>
      </v-col>
    </v-row>
  </v-container>
</template>

<style scoped>
.search-card {
  border: 1px solid rgba(0, 0, 0, 0.08);
}

.search-header {
  background: linear-gradient(135deg, #2e7d32 0%, #1b5e20 100%);
}

.search-input :deep(.v-field) {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.post-card {
  cursor: pointer;
  transition: all 0.3s ease;
  border: 1px solid rgba(0, 0, 0, 0.08);
}

.post-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12) !important;
}

.post-header {
  background: linear-gradient(to bottom, rgba(255, 255, 255, 0.95), rgba(255, 255, 255, 1));
}

.post-image-container {
  background-color: #f5f5f5;
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 300px;
}

.post-image {
  width: 100%;
}
</style>
