<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { supabase } from '@/utils/supabase.js'
import ShowItemDetails from './ShowItemDetails.vue'
import ChatModal from '../common/ChatModal.vue'

const postsWithUsers = ref([])
const savedPosts = ref([]) // Track saved posts
const userId = ref(null) // Current user's ID
const showSuccessModal = ref(false) // Controls success modal
const selectedPostDetails = ref(null)
const isModalVisible = ref(false)
const showChatModal = ref(false)
const selectedChatPost = ref(null)

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
      avatar_url: post.avatar_url,
      created_at: post.created_at,
      user_id: post.user_id
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

// Open chat modal
const openChat = async (post) => {
  // Determine who the conversation partner is
  // If current user owns the post, we need to find who they're chatting with
  // Otherwise, the partner is the post owner
  
  let partnerId, partnerFirstname, partnerLastname, partnerFullName, partnerAvatar
  
  if (post.user_id === userId.value) {
    // Current user owns this post - need to get the other person's info
    // We'll fetch this from messages to find the conversation partner
    const { data: messages } = await supabase
      .from('messages')
      .select('sender_id, receiver_id')
      .eq('post_id', post.post_id)
      .or(`sender_id.eq.${userId.value},receiver_id.eq.${userId.value}`)
      .limit(1)
    
    if (messages && messages.length > 0) {
      partnerId = messages[0].sender_id === userId.value ? messages[0].receiver_id : messages[0].sender_id
      
      // Fetch partner's info from all posts
      const { data: allPosts } = await supabase.rpc('get_posts_with_user_info')
      const partnerPost = allPosts?.find(p => p.user_id === partnerId)
      
      if (partnerPost) {
        partnerFirstname = partnerPost.firstname
        partnerLastname = partnerPost.lastname
        partnerFullName = partnerPost.full_name
        partnerAvatar = partnerPost.profile_pic || partnerPost.avatar_url
      }
    }
  } else {
    // Partner is the post owner
    partnerId = post.user_id
    partnerFirstname = post.firstname
    partnerLastname = post.lastname
    partnerFullName = post.full_name
    partnerAvatar = post.profile_pic || post.avatar_url
  }
  
  selectedChatPost.value = {
    post_id: post.post_id,
    user_id: partnerId,
    firstname: partnerFirstname,
    lastname: partnerLastname,
    full_name: partnerFullName,
    profile_pic: partnerAvatar,
    item_name: post.item_name
  }
  showChatModal.value = true
}

// Handle auto-popup for incoming messages
const handleNewMessageReceived = async (event) => {
  const newMessage = event.detail;
  
  // Only auto-open if we're NOT the sender
  if (newMessage.sender_id === userId.value) return;
  
  // Fetch all posts to get sender's info
  const { data: postData, error: postError } = await supabase
    .rpc('get_posts_with_user_info')
  
  if (postError) {
    console.error('Error fetching post for auto-popup:', postError);
    return;
  }
  
  // Find the post that matches the message's post_id
  const post = postData.find(p => p.post_id === newMessage.post_id);
  
  // Get sender's info from any of their posts
  const senderPost = postData.find(p => p.user_id === newMessage.sender_id);
  
  if (post && senderPost) {
    // Auto-open chat modal with the sender's info
    selectedChatPost.value = {
      post_id: post.post_id,
      user_id: newMessage.sender_id,
      firstname: senderPost.firstname,
      lastname: senderPost.lastname,
      full_name: senderPost.full_name,
      profile_pic: senderPost.profile_pic || senderPost.avatar_url,
      item_name: post.item_name
    };
    showChatModal.value = true;
  }
}

// Fetch data on component mount
onMounted(async () => {
  await fetchUserId()
  await fetchPostsWithUsers()
  await fetchSavedPosts()
  
  // Listen for new message events for auto-popup
  window.addEventListener('new-message-received', handleNewMessageReceived);
});

onUnmounted(() => {
  window.removeEventListener('new-message-received', handleNewMessageReceived);
});

// Listen for profile updates to refresh posts
window.addEventListener('profile-updated', fetchPostsWithUsers)
</script>

<template>
  <v-container class="post-list-container">
    <!-- Post List -->
    <v-row dense class="post-list-row">
      <v-col class="post-col" cols="12" v-for="post in postsWithUsers" :key="post.post_id">
        <v-card
          class="post-card rounded-xl mb-4 overflow-hidden"
          elevation="2"
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
              prepend-icon="mdi-message-text"
              rounded="lg"
              size="default"
              class="font-weight-medium"
              @click.stop="openChat(post)"
            >
              Contact
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-col>
    </v-row>

    <!-- Success Modal -->
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

    <!-- Chat Modal -->
    <ChatModal
      v-model="showChatModal"
      :post-id="selectedChatPost?.post_id"
      :post-owner-id="selectedChatPost?.user_id"
      :post-owner-name="selectedChatPost?.firstname && selectedChatPost?.lastname ? selectedChatPost.firstname + ' ' + selectedChatPost.lastname : selectedChatPost?.full_name"
      :post-owner-avatar="selectedChatPost?.profile_pic || selectedChatPost?.avatar_url"
      :post-title="selectedChatPost?.item_name"
    />
  </v-container>
</template>

<style scoped>
.post-list-container {
  margin-bottom: 0 !important;
  padding-bottom: 10px;
  z-index: 1;
}

.post-list-row {
  margin-bottom: 0 !important;
  z-index: 1;
}

.post-col{
  z-index: 1;
}

/* Mobile: control overflow and add padding for bottom nav */
@media (max-width: 959px) {
  .post-list-container {
    overflow-y: auto;
    max-height: calc(100vh - 260px);
  }
}

.post-card {
  cursor: pointer;
  transition: all 0.3s ease;
  border: 1px solid rgba(0, 0, 0, 0.08);
  position: relative;
  z-index: 2; /* Keep posts BELOW bottom nav */
  margin-bottom: 16px;
}

.post-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12) !important;
  z-index: 2; /* Even on hover, keep it below bottom nav */
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
