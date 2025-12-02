<script setup>
import { ref, onMounted } from 'vue'
import { supabase } from '@/utils/supabase'
import UsersPost from '@/components/layout/UsersPost.vue' // Adjust the path as necessary
import AlertNotification from '../../common/AlertNotification.vue'
import { requiredValidator } from '../../../utils/validators' // Assuming this is defined correctly
import { useRouter } from 'vue-router'

const router = useRouter()

// Function to navigate
const navigateTo = (routeName) => {
  router.push({ name: routeName })
}

// Manage form action states
const formActionDefault = {
  formSuccessMessage: '',
  formErrorMessage: '',
  formProcess: false
}

// Define component properties here
const firstName = ref('')
const lastName = ref('')
const profile_pic = ref('') // File name of the profile image
const facebook_link = ref('') // Reactive reference for Facebook link
const showEditModal = ref(false) // Control modal visibility
const showModal = ref(false) // Modal visibility
const item_name = ref('') // Item name input
const image = ref(null) // Image file input
const description = ref('') // Description input
const posts = ref([]) // Array to store posts
const full_name = ref('')
const avatar_url = ref('')

const profileUrl = 'https://ndmbunubneumkuadlylz.supabase.co/storage/v1/object/public/images/'

// Reset form fields
const resetForm = () => {
  item_name.value = ''
  image.value = null
  description.value = ''
}

const formAction = ref({ ...formActionDefault })

const handlePost = async () => {
  formAction.value = { ...formActionDefault }

  if (!item_name.value || !image.value || !description.value) {
    formAction.value.formErrorMessage = 'All fields are required.'
    return
  }

  formAction.value.formProcess = true

  let imageUrl = ''
  if (image.value) {
    const { data, error } = await supabase.storage
      .from('items')
      .upload(`public/${image.value.name}`, image.value, { upsert: true })

    if (error) {
      formAction.value.formErrorMessage = 'Failed to upload the image.'
      return
    }
    imageUrl = data?.path
  }

  try {
    const {
      data: { user }
    } = await supabase.auth.getUser()

    const userId = user?.id
    const { data: newPost, error: insertError } = await supabase
      .from('posts')
      .insert([
        {
          item_name: item_name.value,
          image: imageUrl,
          description: description.value,
          user_id: userId
        }
      ])
      .select()

    if (insertError) {
      formAction.value.formErrorMessage = 'Failed to create the post.'
      return
    }

    posts.value.unshift(newPost[0]) // Add the new post
    posts.value = [...posts.value] // Ensure reactivity triggers
    formAction.value.formSuccessMessage = 'Post created successfully!'
    resetForm()
    showModal.value = false
  } catch (error) {
    formAction.value.formErrorMessage = 'Unexpected error during post creation.'
  } finally {
    formAction.value.formProcess = false
  }
}

const handleCancel = () => {
  resetForm()
  formAction.value = { ...formActionDefault } // Reset form action state
  showModal.value = false // Close the modal
}

// Fetch user details from Supabase
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
    profile_pic.value = user?.user_metadata?.profile_pic
    facebook_link.value = user?.user_metadata?.facebook_link || ''
    full_name.value = user?.user_metadata?.full_name
    avatar_url.value = user?.user_metadata?.avatar_url
  }
}

const updateProfile = async () => {
  const {
    data: { user },
    error: userError
  } = await supabase.auth.getUser()

  if (userError || !user) {
    console.error('User is not authenticated or error fetching user:', userError)
    return
  }

  let uploadedFileName = profile_pic.value

  if (profile_pic.value instanceof File) {
    const filePath = `public/${profile_pic.value.name}`

    // Attempt to upload or overwrite the file if it already exists
    const { error: uploadError } = await supabase.storage
      .from('images')
      .upload(filePath, profile_pic.value, {
        upsert: true // Correctly include the upsert option here
      })

    if (uploadError) {
      console.error('Error uploading image:', uploadError)
      return
    }

    // Store the full path for consistency with how images are displayed
    uploadedFileName = filePath
    console.log('Image uploaded successfully!')
  }

  const updatedMetadata = {
    firstname: firstName.value,
    lastname: lastName.value,
    profile_pic: uploadedFileName,
    facebook_link: facebook_link.value
  }

  const { error: updateUserError } = await supabase.auth.updateUser({ data: updatedMetadata })
  if (updateUserError) {
    console.error('Error updating user metadata in auth.users:', updateUserError)
    return
  }

  const { error: updateRawMetadataError } = await supabase
    .from('auth.users')
    .update({
      raw_user_meta_data: updatedMetadata
    })
    .eq('id', user.id)

  if (updateRawMetadataError && Object.keys(updateRawMetadataError).length > 0) {
    console.error('Error updating raw_user_meta_data in auth.users:', updateRawMetadataError)
  }

  const { error: updatePostsError } = await supabase
    .from('posts')
    .update(updatedMetadata)
    .eq('user_id', user.id)

  if (updatePostsError) {
    console.error('Error updating posts table:', updatePostsError)
  }

  // Update local state immediately
  firstName.value = updatedMetadata.firstname
  lastName.value = updatedMetadata.lastname
  profile_pic.value = updatedMetadata.profile_pic
  facebook_link.value = updatedMetadata.facebook_link

  formAction.value.formSuccessMessage = 'Profile Updated Successfully'
  showEditModal.value = false
  
  // Fetch updated details to ensure everything is in sync
  await fetchUserDetails()
  
  // Emit custom event to notify other components to refresh
  window.dispatchEvent(new CustomEvent('profile-updated'))
}

onMounted(() => {
  fetchUserDetails()
})
</script>

<template>
  <br />
  <br />

  <v-row justify="center">
    <v-col cols="12" sm="12" md="8">
      <v-card class="rounded-xl mb-4 overflow-hidden" max-width="1000" elevation="4">
        <!-- Gradient Header -->
        <div class="profile-header">
          <div class="profile-gradient"></div>
        </div>
        <v-list class="text-center profile-content">
          <div class="profile-section">
            <v-avatar size="150" class="mx-auto profile-avatar" color="white">
              <!-- If profile_pic exists and is not null or empty, or if it's a file name -->
              <v-img
                v-if="profile_pic && profile_pic !== '' && profile_pic !== null"
                :src="profile_pic.startsWith('http') ? profile_pic : profileUrl + profile_pic"
                alt="User Avatar"
                class="mx-auto"
                height="200"
                width="200"
              />

              <!-- Fallback image if profile_pic is not provided or is invalid -->
              <v-img
                v-else
                :src="avatar_url"
                alt="User Avatar"
                class="mx-auto"
                height="200"
                width="200"
              />
            </v-avatar>
            <h2 class="text-center font-weight-bold mt-4 mb-1 text-h5 text-green-darken-3">
              {{ firstName && lastName ? firstName + ' ' + lastName : full_name }}
            </h2>
            <p class="text-caption text-grey-darken-1 mb-4">Student Profile</p>
          </div>
        </v-list>

        <!-- Center the Facebook button -->
        <v-card-actions class="text-center justify-center pb-2">
          <v-btn
            class="rounded-pill bg-light-green-darken-3"
            icon="mdi-facebook"
            :href="facebook_link"
            target="_blank"
            rel="noopener"
          >
          </v-btn>
          <!-- For activity log -->
          <v-btn
            class="rounded-pill bg-light-green-darken-3"
            icon="mdi-history"
            @click="navigateTo('activity')"
          >
          </v-btn>
        </v-card-actions>

        <!-- Other buttons, already centered -->
        <v-card-actions class="mx-auto">
          <v-btn class="rounded-pill bg-light-green-darken-3" block @click="showEditModal = true">
            Edit Profile
          </v-btn>
        </v-card-actions>
        <v-card-actions class="mx-auto">
          <v-btn class="rounded-pill bg-light-green-darken-3" block @click="showModal = true">
            Post Now!
          </v-btn>
        </v-card-actions>
      </v-card>
      <AlertNotification
        :form-success-message="formAction.formSuccessMessage"
        :form-error-message="formAction.formErrorMessage"
      ></AlertNotification>
    </v-col>
  </v-row>

  <!-- Modal for editing profile -->
  <v-dialog v-model="showEditModal" max-width="500px">
    <v-card class="rounded-xl">
      <div class="modal-gradient pa-6">
        <h2 class="text-h5 font-weight-bold text-white text-center">Edit Profile</h2>
      </div>
      <v-card-text class="pt-6 pb-4 px-6">
        <v-form>
          <v-text-field 
            v-model="firstName" 
            label="First Name" 
            variant="outlined" 
            density="comfortable"
            class="mb-2"
          />
          <v-text-field 
            v-model="lastName" 
            label="Last Name" 
            variant="outlined" 
            density="comfortable"
            class="mb-2"
          />
          <v-file-input
            v-model="profile_pic"
            label="Upload Profile Image"
            accept="image/*"
            variant="outlined"
            density="comfortable"
            prepend-icon="mdi-camera"
            class="mb-2"
          />
          <v-text-field
            v-model="facebook_link"
            label="Facebook Profile Link"
            variant="outlined"
            density="comfortable"
            prepend-inner-icon="mdi-facebook"
          />
        </v-form>
      </v-card-text>
      <v-card-actions class="px-6 pb-6">
        <v-btn 
          variant="outlined" 
          color="grey-darken-1" 
          class="flex-grow-1 rounded-pill text-none"
          @click="showEditModal = false"
        >
          Cancel
        </v-btn>
        <v-btn 
          variant="flat" 
          color="green-darken-2" 
          class="flex-grow-1 rounded-pill text-none"
          @click="updateProfile"
        >
          Save Changes
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>

  <!-- Modal for Create Post -->
  <v-dialog v-model="showModal" max-width="500px">
    <v-card class="rounded-xl">
      <div class="modal-gradient pa-6">
        <h2 class="text-h5 font-weight-bold text-white text-center">Create Post</h2>
      </div>
      <v-card-text class="pt-6 pb-4 px-6">
        <v-form>
          <v-text-field
            v-model="item_name"
            label="Item Name"
            variant="outlined"
            density="comfortable"
            class="mb-2"
            :rules="[requiredValidator]"
          />
          <v-file-input
            v-model="image"
            label="Upload Image"
            accept="image/*"
            variant="outlined"
            density="comfortable"
            prepend-icon="mdi-camera"
            class="mb-2"
            :rules="[requiredValidator]"
          />
          <v-textarea
            v-model="description"
            label="Description"
            variant="outlined"
            density="comfortable"
            rows="3"
            :rules="[requiredValidator]"
          />
        </v-form>
      </v-card-text>
      <v-card-actions class="px-6 pb-6">
        <v-btn 
          variant="outlined" 
          color="grey-darken-1" 
          class="flex-grow-1 rounded-pill text-none"
          @click="handleCancel"
        >
          Cancel
        </v-btn>
        <v-btn 
          :disabled="formAction.formProcess" 
          variant="flat" 
          color="green-darken-2" 
          class="flex-grow-1 rounded-pill text-none"
          @click="handlePost"
        >
          Post
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>

  <!-- Posts section -->
  <UsersPost />
</template>

<style scoped>
.profile-header {
  position: relative;
  height: 120px;
  overflow: hidden;
}

.profile-gradient {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(135deg, #2e7d32 0%, #1b5e20 100%);
}

.profile-content {
  margin-top: -60px;
  position: relative;
  z-index: 2;
}

.profile-avatar {
  border: 4px solid white;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.modal-gradient {
  background: linear-gradient(135deg, #2e7d32 0%, #1b5e20 100%);
}

.gap-3 {
  gap: 12px;
}
</style>
