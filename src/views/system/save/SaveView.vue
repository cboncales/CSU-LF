<script setup>
import { ref, onMounted } from 'vue'
import { supabase } from '@/utils/supabase'
import DashboardLayout from '@/components/layout/DashboardLayout.vue'
import SaveLayout from '@/components/layout/save/SaveLayout.vue'
import ShowItemDetails from '../../../components/layout/ShowItemDetails.vue'

const selectedPostDetails = ref(null)
const isModalVisible = ref(false)
const savedPosts = ref([])
const profileUrl = 'https://ndmbunubneumkuadlylz.supabase.co/storage/v1/object/public/images/'

// Show details of a specific post
const showDetails = (post) => {
  selectedPostDetails.value = post // Set the selected post details
  isModalVisible.value = true // Open the modal
}

// Fetching saved posts from the materialized view
const fetchSavedPosts = async () => {
  try {
    const {
      data: { user },
      error: userError
    } = await supabase.auth.getUser()

    if (userError || !user) {
      console.error('User not logged in or error fetching user:', userError?.message)
      return
    }

    // Query saved_posts table and join with posts to get full post details
    const { data: savedPostsData, error: savedError } = await supabase
      .from('saved_posts')
      .select('post_id')
      .eq('user_id', user.id)

    if (savedError) {
      console.error('Error fetching saved posts:', savedError.message)
      return
    }

    if (savedPostsData.length === 0) {
      savedPosts.value = []
      return
    }

    // Get full post details using RPC function
    const { data: postsData, error: postsError } = await supabase.rpc('get_posts_with_user_info')

    if (postsError) {
      console.error('Error fetching posts details:', postsError.message)
      return
    }

    // Filter to only include saved posts
    const savedPostIds = savedPostsData.map(sp => sp.post_id)
    savedPosts.value = postsData
      .filter(post => savedPostIds.includes(post.post_id))
      .map((post) => ({
        post_id: post.post_id,
        item_name: post.item_name,
        description: post.description,
        image: post.image,
        created_at: post.created_at,
        post_owner: {
          first_name: post.firstname,
          last_name: post.lastname,
          profile_pic: post.profile_pic,
          facebook_link: post.facebook_link,
          full_name: post.full_name,
          avatar_url: post.avatar_url
        }
      }))

    console.log('Saved posts fetched:', savedPosts.value)
  } catch (err) {
    console.error('Unexpected error fetching saved posts:', err.message)
  }
}

// Method to remove a post from saved
const removeFromSaved = async (postId) => {
  try {
    const {
      data: { user }
    } = await supabase.auth.getUser()

    if (!user) {
      console.error('User not logged in')
      return
    }

    // Delete the post from saved_posts table
    const { error } = await supabase
      .from('saved_posts')
      .delete()
      .eq('post_id', postId)
      .eq('user_id', user.id)

    if (error) {
      console.error('Error removing post from saved:', error.message)
    } else {
      // Update the local list of saved posts to reflect the removal
      savedPosts.value = savedPosts.value.filter((post) => post.post_id !== postId)
      console.log(`Post with ID ${postId} removed from saved posts.`)
    }
  } catch (err) {
    console.error('Unexpected error while removing post from saved:', err.message)
  }
}

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

// Fetch data on component mount
onMounted(fetchSavedPosts)
</script>

<template>
  <DashboardLayout>
    <template #content>
      <v-container>
        <SaveLayout />
        
        <!-- Empty State -->
        <v-row v-if="savedPosts.length === 0" justify="center" class="mt-8">
          <v-col cols="12" class="text-center">
            <v-img src="/images/empty.svg" max-width="400" class="mx-auto mb-4"></v-img>
            <h3 class="text-h5 text-grey-darken-1 mb-2">No Saved Posts Yet</h3>
            <p class="text-body-2 text-grey">Save posts to view them here later</p>
          </v-col>
        </v-row>

        <!-- Saved Posts -->
        <v-row v-else dense>
          <v-col cols="12" v-for="post in savedPosts" :key="post.post_id">
            <v-card
              class="post-card rounded-xl mb-4 overflow-hidden"
              elevation="3"
              @click="showDetails(post)"
            >
              <!-- Post Header -->
              <div class="post-header pa-4 d-flex align-center">
                <v-avatar size="56" class="mr-3 elevation-2">
                  <v-img
                    v-if="post.post_owner.profile_pic && post.post_owner.profile_pic !== ''"
                    :src="
                      post.post_owner.profile_pic.startsWith('http')
                        ? post.post_owner.profile_pic
                        : profileUrl + post.post_owner.profile_pic
                    "
                    alt="User Avatar"
                    cover
                  />
                  <v-img
                    v-else
                    :src="post.post_owner.avatar_url || '/images/profile-default.png'"
                    alt="Default Avatar"
                    cover
                  />
                </v-avatar>
                <div>
                  <h3 class="text-green-darken-3 font-weight-bold text-subtitle-1 mb-0">
                    {{
                      post.post_owner.first_name && post.post_owner.last_name
                        ? post.post_owner.first_name + ' ' + post.post_owner.last_name
                        : post.post_owner.full_name
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
                <v-btn
                  color="red-darken-2"
                  variant="flat"
                  prepend-icon="mdi-bookmark-remove"
                  rounded="lg"
                  size="default"
                  class="font-weight-medium"
                  @click.stop="removeFromSaved(post.post_id)"
                >
                  Remove
                </v-btn>
                <v-spacer></v-spacer>
                <v-btn
                  color="orange-darken-2"
                  variant="flat"
                  prepend-icon="mdi-facebook-messenger"
                  rounded="lg"
                  size="default"
                  class="font-weight-medium"
                  :href="post.post_owner.facebook_link"
                  target="_blank"
                  rel="noopener"
                >
                  Send Message
                </v-btn>
              </v-card-actions>
            </v-card>
          </v-col>
        </v-row>

        <!-- Modal for Post Details -->
        <v-dialog v-model="isModalVisible" max-width="600">
          <template v-slot:default>
            <ShowItemDetails :postId="selectedPostDetails?.post_id" />
          </template>
        </v-dialog>
      </v-container>
    </template>
  </DashboardLayout>
</template>

<style scoped>
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