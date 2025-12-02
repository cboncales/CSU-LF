<script setup>
import { ref, onMounted } from 'vue'
import { supabase } from '@/utils/supabase.js'
import ShowItemDetails from './ShowItemDetails.vue'

const postsWithUsers = ref([])
const savedPosts = ref([]) // Track saved posts
const userId = ref(null) // Current user's ID
const showSuccessModal = ref(false) // Controls success modal
const selectedPostDetails = ref(null)
const isModalVisible = ref(false)

// URL of the image
const profileUrl = 'https://ndmbunubneumkuadlylz.supabase.co/storage/v1/object/public/images/'

// Fetch current user ID
const fetchUserId = async () => {
  const { data, error } = await supabase.auth.getUser()
  if (error) {
    console.error('Error fetching user ID:', error.message)
  } else {
    userId.value = data.user?.id
  }
}

// Fetch posts with user info
const fetchPostsWithUsers = async () => {
  try {
    const { data, error } = await supabase.rpc('get_posts_with_user_info') //functions for get_posts_with_user_info
    if (error) {
      console.error('Error fetching posts with user info:', error.message)
      console.error('Full error:', error)
      return
    }

    console.log('Posts data from RPC:', data)
    
    postsWithUsers.value = data.map((post) => ({
      post_id: post.post_id,
      item_name: post.item_name,
      description: post.description,
      image: post.image,
      firstname: post.firstname,
      lastname: post.lastname,
      facebook_link: post.facebook_link,
      profile_pic: post.profile_pic,
      full_name: post.full_name,
      avatar_url: post.avatar_url
    }))
    
    console.log('Mapped posts:', postsWithUsers.value)
  } catch (err) {
    console.error('Unexpected error fetching posts with user info:', err.message)
  }
}

// Fetch saved posts
const fetchSavedPosts = async () => {
  if (!userId.value) return

  const { data, error } = await supabase
    .from('saved_posts')
    .select('post_id')
    .eq('user_id', userId.value)

  if (error) {
    console.error('Error fetching saved posts:', error.message)
  } else {
    savedPosts.value = data.map((saved) => saved.post_id)
  }
}

// Check if a post is already saved
const isSaved = (postId) => {
  return savedPosts.value.includes(postId)
}

// Handle saving a post
const savePost = async (postId) => {
  if (!userId.value) {
    toast.error('You must be logged in to save a post.')
    return
  }

  if (isSaved(postId)) {
    toast.info('This post is already saved.')
    return
  }

  const { error } = await supabase
    .from('saved_posts')
    .insert([{ post_id: postId, user_id: userId.value }])

  if (error) {
    toast.error('Error saving post. Please try again.')
    console.error('Error saving post:', error.message)
  } else {
    savedPosts.value.push(postId) // Update saved posts
    showSuccessModal.value = true // Show the success modal
    toast.success('Post saved successfully!')
  }
}

// Show details of a specific post
const showDetails = (post) => {
  selectedPostDetails.value = post // Set the selected post details
  isModalVisible.value = true // Open the modal
}


// Fetch data on component mount
onMounted(async () => {
  await fetchUserId()
  await fetchPostsWithUsers()
  await fetchSavedPosts()
})

// Listen for profile updates to refresh posts
window.addEventListener('profile-updated', fetchPostsWithUsers)
</script>

<template>
  <v-container>
    <!-- Post List -->
    <v-row dense>
      <v-col cols="12" v-for="post in postsWithUsers" :key="post.post_id">
        <v-card
          class="post-card rounded-xl mb-4 overflow-hidden"
          elevation="3"
          @click="showDetails(post)"
        >
          <!-- Post Header -->
          <div class="post-header pa-4 d-flex align-center">
            <v-avatar size="56" class="mr-3 elevation-2">
              <v-img
                v-if="post.profile_pic && post.profile_pic !== ''"
                :src="post.profile_pic.startsWith('http') ? post.profile_pic : profileUrl + post.profile_pic"
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
                {{ post.firstname && post.lastname ? post.firstname + ' ' + post.lastname : post.full_name }}
              </h3>
              <p class="text-caption text-grey-darken-1 mb-0">Found an item</p>
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
              :color="isSaved(post.post_id) ? 'grey' : 'green-darken-2'"
              :disabled="isSaved(post.post_id)"
              variant="flat"
              prepend-icon="mdi-bookmark"
              rounded="lg"
              size="default"
              class="font-weight-medium"
              @click.stop="savePost(post.post_id)"
            >
              {{ isSaved(post.post_id) ? 'Saved' : 'Save' }}
            </v-btn>
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
              Contact Owner
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-col>
    </v-row>    <!-- Success Modal -->
    <v-dialog v-model="showSuccessModal" persistent max-width="400">
      <v-card>
        <v-card-title class="headline text-light-green-darken-3">Success</v-card-title>
        <v-card-text class="text-light-green-darken-3"
          >Your post has been saved successfully!</v-card-text
        >
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="green darken-1" text @click="showSuccessModal = false">Close</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

            <!-- Modal for Post Details to resolve the undefined post-id -->
        <v-dialog v-model="isModalVisible" max-width="600">
          <template v-slot:default>
            <!-- Pass postId from selectedPostDetails to ShowItemDetails -->
            <ShowItemDetails :postId="selectedPostDetails?.post_id" />
          </template>
        </v-dialog>
  </v-container>
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
