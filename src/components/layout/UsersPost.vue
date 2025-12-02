<script setup>
import { ref, onMounted } from 'vue'
import ShowItemDetails from './ShowItemDetails.vue'
import AlertNotification from '../common/AlertNotification.vue'
import { supabase } from '@/utils/supabase.js'

const posts = ref([]) // Array to store posts
const firstName = ref('') // First name of the user
const lastName = ref('') // Last name of the user
const profilePic = ref('') // Profile picture of the user
const selectedPostDetails = ref(null)
const isModalVisible = ref(false)
const full_name = ref('')
const avatar_url = ref('')

// URL for fetching the image
const profileUrl = 'https://ndmbunubneumkuadlylz.supabase.co/storage/v1/object/public/images/'

// Show details of a specific post
const showDetails = (post) => {
  selectedPostDetails.value = post // Set the selected post details
  isModalVisible.value = true // Open the modal
}

// Manage form action states
const formActionDefault = {
  formSuccessMessage: '',
  formErrorMessage: '',
  formProcess: false
}

const formAction = ref({ ...formActionDefault })

// Fetch user's details and posts
const fetchUserData = async () => {
  const {
    data: { user },
    error: authError
  } = await supabase.auth.getUser()

  if (authError || !user) {
    console.error('Error fetching user details:', authError)
    return
  }

  // Fetch the profile data directly from user metadata (real-time)
  firstName.value = user.user_metadata?.firstname
  lastName.value = user.user_metadata?.lastname
  profilePic.value = user.user_metadata?.profile_pic
  full_name.value = user?.user_metadata?.full_name
  avatar_url.value = user?.user_metadata?.avatar_url

  // Fetch the user's posts
  const { data: postsData, error: fetchError } = await supabase
    .from('posts')
    .select()
    .eq('user_id', user.id) // Filter posts by the logged-in user

  if (fetchError) {
    console.error('Error fetching user posts:', fetchError)
  } else {
    posts.value = postsData // Store user's posts
    console.log('User posts fetched:', posts.value)
  }
}

const deletePost = async (post) => {
  // Confirm deletion
  const confirmed = confirm('Are you sure you want to delete this post?');
  if (!confirmed) return;

  // Attempt to delete the post from the database
  const { error } = await supabase.from('posts').delete().eq('id', post.id);

  if (error) {
    console.error('Error deleting post:', error);
    formAction.formErrorMessage = 'Post not deleted';
  } else {
    // Remove the post from the `posts` array to update the UI
    posts.value = posts.value.filter((p) => p.id !== post.id);
    console.log('Post deleted successfully');
    formAction.value.formSuccessMessage = 'Successfully Deleted';
  }
};


onMounted(fetchUserData) // Fetch user data and posts when the component is mounted

// Listen for profile updates from other components
window.addEventListener('profile-updated', fetchUserData)

///edit the post
const editPostData = ref(null) // Holds the post data to edit
const isEditModalVisible = ref(false) // Controls the visibility of the edit modal

const editPost = (post) => {
  editPostData.value = { ...post } // Load the post data into the form
  isEditModalVisible.value = true // Show the modal
}

const uploadImage = async (file) => {
  try {
    const { data, error } = await supabase.storage
      .from('items') // Your Supabase storage bucket
      .upload(`public/${file.name}`, file, {
        cacheControl: '3600',
        upsert: true // Prevent overwriting existing files
      })

    if (error) {
      throw error
    }

    // Return the file path on successful upload
    return data.path
  } catch (error) {
    console.error('Error uploading image:', error)
    return null
  }
}

  const updatePost = async () => {
  formAction.value = { ...formActionDefault, formProcess: true }

  // Check if a new image is selected
  if (editPostData.value.imageFile) {
    const uploadedImagePath = await uploadImage(editPostData.value.imageFile)

    if (!uploadedImagePath) {
      formAction.value.formErrorMessage = 'Image upload failed'
      formAction.value.formProcess = false
      return
    }

    editPostData.value.image = uploadedImagePath // Use uploaded image path
  }

  const { id, image, item_name, description } = editPostData.value

  const { error } = await supabase
    .from('posts')
    .update({ image, item_name, description })
    .eq('id', id)

  if (error) {
    console.error('Error updating post:', error)
    formAction.formErrorMessage = 'Post Not Updated'
  } else {
    // Update the local data
    const index = posts.value.findIndex((p) => p.id === id)
    if (index !== -1) {
      posts.value[index] = { ...editPostData.value }
    }
    console.log('Post updated successfully')
    formAction.value.formSuccessMessage = 'Post Updated Successfully '
    isEditModalVisible.value = false // Close the modal
  }

  formAction.value.formProcess = false
}

</script>

<template>
  <!-- Alert Notification -->
  <AlertNotification
    :form-success-message="formAction.formSuccessMessage"
    :form-error-message="formAction.formErrorMessage"
  ></AlertNotification>
  <v-container>
    <!-- Display message when there are no posts and have -->
    <v-row justify="center" align="center" class="">
      <v-col cols="8" class="text-center">
        <v-divider class="">
          <p v-if="posts.length === 0">No Posts</p>
          <p v-else>Your Posts</p>
        </v-divider>
      </v-col>
    </v-row>
    <v-row dense>
      <v-col cols="12" v-for="post in posts" :key="post.id">
        <v-card
          class="mb-4 rounded-xl post-card"
          outlined
          elevation="3"
          link
          @click="showDetails(post)"
        >
          <v-list-item>
            <!-- Poster Image -->
            <v-row align="center" class="mx-1 my-2">
              <!-- Avatar Column -->
              <v-col cols="auto" class="d-flex align-center">
                <v-avatar size="56" color="grey-lighten-2">
                  <!-- Check if profile_pic exists and is not null or empty -->
                  <v-img
                    v-if="profilePic && profilePic !== ''"
                    :src="profilePic.startsWith('http') ? profilePic : profileUrl + profilePic"
                    alt="User Avatar and default profile"
                    class="mx-auto"
                    height="200"
                    width="200"
                  />

                  <!-- Fallback image if profile_pic is not available -->
                  <v-img
                    v-else
                    :src="avatar_url || 'default-avatar-url.png'"
                    alt="google profile"
                    class="mx-auto"
                    height="200"
                    width="200"
                  />
                </v-avatar>
              </v-col>
              <!-- Name Column -->
              <v-col class="d-flex flex-column justify-center">
                <h3 class="text-subtitle-1 font-weight-bold text-green-darken-3 mb-0">
                  {{ firstName && lastName ? firstName + ' ' + lastName : full_name }}
                </h3>
                <p class="text-caption text-grey-darken-1 mb-0">Posted an item</p>
              </v-col>
              <!-- Menu Column -->
              <v-col cols="auto" class="d-flex align-center justify-end">
                <v-menu offset-y>
                  <template v-slot:activator="{ props }">
                    <v-icon icon="mdi-format-list-bulleted" class="ml-auto" v-bind="props"></v-icon>
                  </template>
                  <v-list>
                    <v-list-item @click="editPost(post)">
                      <v-list-item-title>Edit Post</v-list-item-title>
                    </v-list-item>
                    <v-list-item @click="deletePost(post)">
                      <v-list-item-title>Delete Post</v-list-item-title>
                    </v-list-item>
                  </v-list>
                </v-menu>
              </v-col>
            </v-row>
          </v-list-item>
          <!-- Post Image -->
          <div class="post-image-container" v-if="post.image">
            <v-img
              :src="`https://ndmbunubneumkuadlylz.supabase.co/storage/v1/object/public/items/${post.image}`"
              contain
              :alt="post.item_name || 'Post Image'"
            />
          </div>
          <v-card-title class="text-green-darken-3 font-weight-bold">{{ post.item_name }}</v-card-title>
          <v-card-subtitle class="text-grey-darken-2">{{
            post.description
          }}</v-card-subtitle>
        </v-card>
      </v-col>
    </v-row>

    <!-- Modal for Post Details -->
    <v-dialog v-model="isModalVisible" max-width="600">
      <template v-slot:default>
        <!-- Pass postId from selectedPostDetails to ShowItemDetails -->
        <ShowItemDetails :postId="selectedPostDetails?.id" />
      </template>
    </v-dialog>

    <!-- Edit Post Modal -->
    <v-dialog v-model="isEditModalVisible" max-width="600">
      <template v-slot:default>
        <v-card class="rounded-xl">
          <div class="modal-gradient pa-6">
            <h2 class="text-h5 font-weight-bold text-white text-center">Edit Post</h2>
          </div>
          <v-card-text class="pt-6 pb-4 px-6">
            <v-form>
              <v-text-field
                v-model="editPostData.item_name"
                label="Item Name"
                variant="outlined"
                density="comfortable"
                class="mb-2"
              ></v-text-field>
              <v-textarea
                v-model="editPostData.description"
                label="Description"
                variant="outlined"
                density="comfortable"
                rows="3"
                class="mb-2"
              ></v-textarea>
              <v-file-input
                label="Upload Image"
                v-model="editPostData.imageFile"
                accept="image/*"
                variant="outlined"
                density="comfortable"
                prepend-icon="mdi-camera"
                show-size
              ></v-file-input>
            </v-form>
          </v-card-text>
          <v-card-actions class="px-6 pb-6">
            <v-btn 
              variant="outlined" 
              color="grey-darken-1" 
              class="flex-grow-1 rounded-pill text-none"
              @click="isEditModalVisible = false"
            >
              Cancel
            </v-btn>
            <v-btn 
              variant="flat" 
              color="green-darken-2" 
              class="flex-grow-1 rounded-pill text-none"
              @click="updatePost"
            >
              Save Changes
            </v-btn>
          </v-card-actions>
        </v-card>
      </template>
    </v-dialog>
  </v-container>
</template>

<style scoped>
.post-card {
  transition: all 0.3s ease;
  cursor: pointer;
}

.post-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15) !important;
}

.post-image-container {
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #f5f5f5;
  min-height: 300px;
  max-height: 500px;
  overflow: hidden;
}

.modal-gradient {
  background: linear-gradient(135deg, #2e7d32 0%, #1b5e20 100%);
}
</style>
