<script setup>
import { defineProps, defineEmits, ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { supabase } from '@/utils/supabase'
import { useAuthStore } from '@/stores/authUser'


//profile url in supabase
const profileUrl = 'https://ndmbunubneumkuadlylz.supabase.co/storage/v1/object/public/images/'

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

const emit = defineEmits(['update:modelValue'])

// Router and navigation logic
const router = useRouter()
const currentRoute = ref(router.currentRoute.value.name)

const navigateTo = (routeName) => {
  router.push({ name: routeName })
  currentRoute.value = routeName
}

const onLogout = async () => {
  await supabase.auth.signOut()
  const authStore = useAuthStore()
  authStore.logout()
  router.replace('/login')
}

// Fetch user details from user_profiles_view
const full_name = ref('')
const avatar_url = ref('')
const firstName = ref('')
const lastName = ref('')
const profile_pic = ref('')

const fetchUserDetails = async () => {
  // Get the current authenticated user directly from auth
  const { data: { user }, error: authError } = await supabase.auth.getUser();
  
  if (authError) {
    console.error(authError);
    return;
  }

  // Get data directly from user metadata (real-time, no caching)
  full_name.value = user?.user_metadata?.full_name;
  avatar_url.value = user?.user_metadata?.avatar_url;
  firstName.value = user?.user_metadata?.firstname;
  lastName.value = user?.user_metadata?.lastname;
  profile_pic.value = user?.user_metadata?.profile_pic;
};


onMounted(fetchUserDetails)

// Listen for profile updates
window.addEventListener('profile-updated', fetchUserDetails)
</script>

<template>
  <div class="side-nav-card rounded-xl shadow-lg overflow-hidden h-full">
    <!-- Profile Section -->
    <div class="profile-header text-center py-6 px-4">
      <div class="d-flex align-center justify-center mb-3">
        <v-img src="/images/logo.png" max-width="50" contain class="mr-2" />
        <div class="text-left">
          <p class="text-white text-caption font-weight-bold mb-0" style="line-height: 1.2;">
            College of Computing<br>and Information Sciences
          </p>
        </div>
      </div>
      <v-avatar size="80" class="mx-auto mb-2 elevation-4" color="white">
        <v-img
          v-if="profile_pic && typeof profile_pic === 'string' && profile_pic !== '' && profile_pic !== null"
          :src="profile_pic.startsWith('http') ? profile_pic : profileUrl + profile_pic"
          alt="User Avatar"
          cover
        />
        <v-img
          v-else
          :src="avatar_url"
          alt="User Avatar"
          cover
        />
      </v-avatar>
      <h3 class="text-white font-weight-bold text-h6 mb-1">
        {{ firstName && lastName ? firstName + ' ' + lastName : full_name }}
      </h3>
    </div>

    <!-- Search Button -->
    <div class="px-3 mb-2 mt-2">
      <v-btn
        @click="navigateTo('search')"
        prepend-icon="mdi-magnify"
        block
        rounded="lg"
        color="white"
        variant="flat"
        class="text-green-darken-3 font-weight-bold shadow-md"
      >
        Search
      </v-btn>
    </div>

    <!-- Navigation Menu -->
    <v-list class="bg-transparent px-2" color="transparent">
      <v-list-item
        @click="navigateTo('dashboard')"
        :class="{ 'nav-item-active': currentRoute === 'dashboard' }"
        class="nav-item rounded-lg mb-1"
        prepend-icon="mdi-home"
      >
        <v-list-item-title class="text-white font-weight-medium">Home</v-list-item-title>
      </v-list-item>
      
      <v-list-item
        @click="navigateTo('save')"
        :class="{ 'nav-item-active': currentRoute === 'save' }"
        class="nav-item rounded-lg mb-1"
        prepend-icon="mdi-bookmark"
      >
        <v-list-item-title class="text-white font-weight-medium">Saved</v-list-item-title>
      </v-list-item>
      
      <v-list-item
        @click="navigateTo('profile')"
        :class="{ 'nav-item-active': currentRoute === 'profile' }"
        class="nav-item rounded-lg mb-1"
        prepend-icon="mdi-account"
      >
        <v-list-item-title class="text-white font-weight-medium">Profile</v-list-item-title>
      </v-list-item>
      
      <v-list-item
        @click="navigateTo('about')"
        :class="{ 'nav-item-active': currentRoute === 'about' }"
        class="nav-item rounded-lg mb-1"
        prepend-icon="mdi-information"
      >
        <v-list-item-title class="text-white font-weight-medium">About CSULF</v-list-item-title>
      </v-list-item>
    </v-list>

    <!-- Logout Button -->
    <div class="pa-4 mb-4 mt-auto">
      <v-btn
        @click="onLogout"
        prepend-icon="mdi-logout"
        block
        rounded="lg"
        color="white"
        variant="flat"
        class="text-green-darken-3 font-weight-bold shadow-md"
      >
        Sign Out
      </v-btn>
    </div>
  </div>
</template>

<style scoped>
.side-nav-card {
  background: linear-gradient(180deg, #2e7d32 0%, #1b5e20 100%);
  display: flex;
  flex-direction: column;
}

.profile-header {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.2);
}

.nav-item {
  cursor: pointer;
  transition: all 0.2s ease;
}

.nav-item:hover {
  background-color: rgba(255, 255, 255, 0.15);
}

.nav-item-active {
  background-color: rgba(255, 255, 255, 0.25) !important;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}
</style>
