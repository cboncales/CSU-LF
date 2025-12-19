<script setup>
import { ref, onMounted } from 'vue'
import { supabase } from '@/utils/supabase' // Ensure this points to your Supabase instance
import AlertNotification from '../common/AlertNotification.vue'
import { requiredValidator } from '../../utils/validators' // Assuming this is defined correctly

// Reactive references for form and modal
const showModal = ref(false) // Modal visibility
const item_name = ref('') // Item name input
const image = ref(null) // Image file input
const description = ref('') // Description input
const posts = ref([]) // Array to store posts
const firstName = ref('') // User's first name
const lastName = ref('') // User's last name
const full_name = ref('')
const avatar_url = ref('')

// Manage form action states
const formActionDefault = {
  formSuccessMessage: '',
  formErrorMessage: '',
  formProcess: false
}
const formAction = ref({ ...formActionDefault })

// Reset form fields
const resetForm = () => {
  item_name.value = ''
  image.value = null
  description.value = ''
}

// Handle Post Submission
const handlePost = async () => {
  // Reset messages
  formAction.value = { ...formActionDefault }

  // Validate required fields
  if (!item_name.value || !image.value || !description.value) {
    formAction.value.formErrorMessage = 'All fields are required.'
    console.error('Validation error: Some fields are empty.')
    return
  }

  formAction.value.formProcess = true // Indicate process start

  let imageUrl = ''
  if (image.value) {
    try {
      // Attempt to upload or overwrite the file if it already exists
      const { data, error } = await supabase.storage
        .from('items')
        .upload(`public/${image.value.name}`, image.value, {
          upsert: true // Pass upsert as an option here
        })

      if (error) {
        console.error('Image upload error:', error)
        formAction.value.formErrorMessage = 'Failed to upload the image.'
        return
      }
      imageUrl = data?.path
      console.log('Image uploaded successfully:', imageUrl)
    } catch (error) {
      console.error('Error uploading image:', error)
      formAction.value.formErrorMessage = 'Unexpected error during image upload.'
      return
    }
  }

  try {
    const {
      data: { user },
      error: authError
    } = await supabase.auth.getUser()

    if (authError) {
      console.error('Auth error:', authError)
      formAction.value.formErrorMessage = 'Failed to retrieve user information.'
      return
    }

    const userId = user?.id

    const { error: insertError } = await supabase.from('posts').insert([
      {
        item_name: item_name.value,
        image: imageUrl,
        description: description.value,
        user_id: userId
      }
    ])

    if (insertError) {
      console.error('Insert error:', insertError)
      formAction.value.formErrorMessage = 'Failed to create the post.'
      return
    }

    formAction.value.formSuccessMessage = 'Post created successfully!'

    // Fetch the new list of posts, filtering by userId for UsersPost.vue
    const { data: postsData, error: fetchError } = await supabase
      .from('posts')
      .select()
      .eq('user_id', userId) // Fetch only the logged-in user's posts

    if (fetchError) {
      console.error('Fetch error:', fetchError)
      formAction.value.formErrorMessage = 'Failed to fetch updated posts.'
      return
    }

    posts.value = postsData // Update posts with user's posts
    console.log('Posts fetched:', posts.value)

    resetForm() // Clear the form fields
    showModal.value = false // Close the modal
  } catch (error) {
    console.error('Error during post creation:', error)
    formAction.value.formErrorMessage = 'Unexpected error during post creation.'
  } finally {
    formAction.value.formProcess = false // Indicate process end
  }
}

const handleCancel = () => {
  resetForm()
  formAction.value = { ...formActionDefault } // Reset form action state
  showModal.value = false // Close the modal
}

// Fetch user details on component mount
const fetchUserDetails = async () => {
  const {
    data: { user },
    error
  } = await supabase.auth.getUser()
  if (error) {
    console.error(error)
  } else {
    firstName.value = user?.user_metadata?.firstname
    lastName.value = user?.user_metadata?.lastname
    full_name.value = user?.user_metadata?.full_name
    avatar_url.value = user?.user_metadata?.avatar_url
  }
}

onMounted(() => {
  fetchUserDetails()
})
</script>

<template>
  <v-container>
    <v-row justify="center">
      <v-col cols="12">
        <v-card
          class="create-post-card rounded-xl overflow-hidden"
          elevation="2"
        >
          <div class="pa-5">
            <div class="d-flex align-center mb-3">
              <v-icon size="32" color="green-darken-3" class="mr-3">mdi-lightbulb-on</v-icon>
              <div>
                <h2 class="text-h6 text-green-darken-3 font-weight-bold mb-0">
                  Hello {{ firstName && lastName ? firstName + ' ' + lastName : full_name || 'Guest' }}!
                </h2>
                <p class="text-body-2 text-grey-darken-1 mb-0">Lost or Found something? Help others find their belongings</p>
              </div>
            </div>
            <v-btn
              class="rounded-lg font-weight-bold text-white"
              color="green-darken-3"
              size="large"
              prepend-icon="mdi-plus-circle"
              block
              elevation="0"
              @click="showModal = true"
            >
              Create Post
            </v-btn>
          </div>
        </v-card>
      </v-col>
    </v-row>

    <!-- Modal for Create Post -->
    <v-dialog v-model="showModal" max-width="600px" persistent>
      <v-card class="rounded-xl overflow-hidden">
        <!-- Header -->
        <div class="modal-header pa-5">
          <div class="d-flex align-center">
            <v-icon size="32" color="white" class="mr-3">mdi-plus-circle-outline</v-icon>
            <div>
              <h2 class="text-h5 text-white font-weight-bold mb-0">Create New Post</h2>
              <p class="text-body-2 text-white text-opacity-90 mb-0">Share a found item with the community</p>
            </div>
          </div>
        </div>

        <!-- Form Content -->
        <v-card-text class="pa-6">
          <v-form>
            <div class="mb-4">
              <label class="text-body-2 font-weight-medium text-grey-darken-2 mb-2 d-block">Item Name</label>
              <v-text-field
                v-model="item_name"
                placeholder="What did you lost/find?"
                variant="outlined"
                density="comfortable"
                rounded="lg"
                color="green-darken-3"
                :rules="[requiredValidator]"
                hide-details="auto"
              />
            </div>

            <div class="mb-4">
              <label class="text-body-2 font-weight-medium text-grey-darken-2 mb-2 d-block">Upload Image</label>
              <v-file-input
                v-model="image"
                placeholder="Choose an image"
                accept="image/*"
                variant="outlined"
                density="comfortable"
                rounded="lg"
                color="green-darken-3"
                prepend-icon=""
                prepend-inner-icon="mdi-camera"
                :rules="[requiredValidator]"
                hide-details="auto"
              />
            </div>

            <div class="mb-2">
              <label class="text-body-2 font-weight-medium text-grey-darken-2 mb-2 d-block">Description</label>
              <v-textarea
                v-model="description"
                placeholder="Describe the item and where you lost/found it..."
                variant="outlined"
                rounded="lg"
                color="green-darken-3"
                rows="4"
                :rules="[requiredValidator]"
                hide-details="auto"
              />
            </div>
          </v-form>
        </v-card-text>

        <!-- Actions -->
        <v-card-actions class="pa-5 pt-0">
          <v-btn
            variant="outlined"
            color="grey-darken-1"
            rounded="lg"
            size="large"
            class="font-weight-medium flex-grow-1"
            @click="handleCancel"
          >
            Cancel
          </v-btn>
          <v-btn
            variant="flat"
            color="green-darken-3"
            rounded="lg"
            size="large"
            class="font-weight-bold text-white flex-grow-1"
            :disabled="formAction.formProcess"
            :loading="formAction.formProcess"
            @click="handlePost"
          >
            Create Post
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>

  <!-- Alert Notification -->
  <AlertNotification
    :form-success-message="formAction.formSuccessMessage"
    :form-error-message="formAction.formErrorMessage"
  </AlertNotification>
</template>

<style scoped>
.modal-header {
  background: linear-gradient(135deg, #2e7d32 0%, #1b5e20 100%);
}
</style>
