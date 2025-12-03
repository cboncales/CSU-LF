<script setup>
import { defineProps, defineEmits, ref, onMounted, onUnmounted } from 'vue'
import { useRouter } from 'vue-router'
import { supabase } from '@/utils/supabase'
import { useAuthStore } from '@/stores/authUser'
import SideNews from './SideNews.vue'


//profile url in supabase
const profileUrl = 'https://ndmbunubneumkuadlylz.supabase.co/storage/v1/object/public/images/'
const unreadMessageCount = ref(0)
const currentUserId = ref(null)
let realtimeChannel = null
const showMobileMenu = ref(false)

// Props
const props = defineProps({
  modelValue: {
    type: Boolean,
    required: false,
    default: true
  },
  permanent: {
    type: Boolean,
    default: false
  },
  mobileOnly: {
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
  currentUserId.value = user?.id;
  
  // Fetch unread messages after getting user ID
  await fetchUnreadCount();
};

// Fetch unread message count
const fetchUnreadCount = async () => {
  if (!currentUserId.value) return;
  
  const { count, error } = await supabase
    .from('messages')
    .select('*', { count: 'exact', head: true })
    .eq('receiver_id', currentUserId.value)
    .eq('is_read', false);
  
  if (error) {
    console.error('Error fetching unread count:', error);
  } else {
    unreadMessageCount.value = count || 0;
  }
};

// Setup realtime subscription for new messages
const setupRealtimeSubscription = () => {
  if (!currentUserId.value) return;
  
  realtimeChannel = supabase
    .channel(`notifications:${currentUserId.value}`)
    .on(
      'postgres_changes',
      {
        event: 'INSERT',
        schema: 'public',
        table: 'messages',
        filter: `receiver_id=eq.${currentUserId.value}`
      },
      (payload) => {
        // Increment unread count
        unreadMessageCount.value++;
        
        // Trigger auto-popup event
        window.dispatchEvent(new CustomEvent('new-message-received', { 
          detail: payload.new 
        }));
      }
    )
    .on(
      'postgres_changes',
      {
        event: 'UPDATE',
        schema: 'public',
        table: 'messages',
        filter: `receiver_id=eq.${currentUserId.value}`
      },
      (payload) => {
        // Refresh unread count when messages are marked as read
        if (payload.new.is_read) {
          fetchUnreadCount();
        }
      }
    )
    .subscribe();
};

// Cleanup realtime subscription
const cleanupRealtimeSubscription = () => {
  if (realtimeChannel) {
    supabase.removeChannel(realtimeChannel);
    realtimeChannel = null;
  }
};

// Listen for message read events from other components
window.addEventListener('messages-read', () => {
  fetchUnreadCount();
});

onMounted(async () => {
  await fetchUserDetails();
  setupRealtimeSubscription();
});

onUnmounted(() => {
  cleanupRealtimeSubscription();
});

// Listen for profile updates
window.addEventListener('profile-updated', fetchUserDetails)
</script>

<template>
  <!-- Desktop Side Navigation -->
  <div v-if="!mobileOnly" class="side-nav-card rounded-xl shadow-lg overflow-hidden h-full d-none d-md-flex">
    <!-- Profile Section -->
    <div class="profile-header text-center py-2 px-4">
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
        @click="navigateTo('messages')"
        :class="{ 'nav-item-active': currentRoute === 'messages' }"
        class="nav-item rounded-lg mb-1"
        prepend-icon="mdi-message-text"
      >
        <v-list-item-title class="text-white font-weight-medium">Messages</v-list-item-title>
        <template v-slot:append v-if="unreadMessageCount > 0">
          <v-badge
            :content="unreadMessageCount"
            color="red"
            inline
            class="message-badge"
          ></v-badge>
        </template>
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
        class="nav-item rounded-lg"
        prepend-icon="mdi-information"
      >
        <v-list-item-title class="text-white font-weight-medium">About CSULF</v-list-item-title>
      </v-list-item>
    </v-list>

    <!-- Logout Button -->
    <div class="pa-4">
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

  <!-- Mobile Bottom Navigation -->
  <v-bottom-navigation
    v-model="currentRoute"
    class="d-md-none mobile-bottom-nav"
    color="green-darken-3"
    grow
    bg-color="white"
    app
  >
    <v-btn value="dashboard" @click="navigateTo('dashboard')">
      <v-icon>mdi-home</v-icon>
      <span class="text-caption">Home</span>
    </v-btn>

    <v-btn value="search" @click="navigateTo('search')">
      <v-icon>mdi-magnify</v-icon>
      <span class="text-caption">Search</span>
    </v-btn>

    <v-btn value="save" @click="navigateTo('save')">
      <v-icon>mdi-bookmark</v-icon>
      <span class="text-caption">Saved</span>
    </v-btn>

    <v-btn value="messages" @click="navigateTo('messages')">
      <v-badge
        v-if="unreadMessageCount > 0"
        :content="unreadMessageCount"
        color="red"
        overlap
      >
        <v-icon>mdi-message-text</v-icon>
      </v-badge>
      <v-icon v-else>mdi-message-text</v-icon>
      <span class="text-caption">Messages</span>
    </v-btn>

    <v-btn value="menu" @click="showMobileMenu = true">
      <v-icon>mdi-menu</v-icon>
      <span class="text-caption">More</span>
    </v-btn>
  </v-bottom-navigation>

  <!-- Mobile Menu Dialog (includes Profile, About, Side News, Logout) -->
  <v-dialog v-model="showMobileMenu" fullscreen transition="dialog-bottom-transition">
    <v-card class="mobile-menu-card">
      <v-toolbar color="green-darken-3" dark>
        <v-toolbar-title>Menu</v-toolbar-title>
        <v-spacer></v-spacer>
        <v-btn icon @click="showMobileMenu = false">
          <v-icon>mdi-close</v-icon>
        </v-btn>
      </v-toolbar>

      <div class="pa-4">
        <!-- Profile Section -->
        <v-card class="mb-4 pa-4 text-center" color="green-lighten-5">
          <v-avatar size="80" class="mx-auto mb-3 elevation-4" color="white">
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
          <h3 class="text-green-darken-3 font-weight-bold text-h6 mb-1">
            {{ firstName && lastName ? firstName + ' ' + lastName : full_name }}
          </h3>
        </v-card>

        <!-- Menu Items -->
        <v-list>
          <v-list-item
            @click="navigateTo('profile'); showMobileMenu = false"
            prepend-icon="mdi-account"
            title="Profile"
            class="mb-2"
          ></v-list-item>

          <v-list-item
            @click="navigateTo('about'); showMobileMenu = false"
            prepend-icon="mdi-information"
            title="About CSULF"
            class="mb-2"
          ></v-list-item>

          <v-divider class="my-3"></v-divider>

          <!-- Side News Section -->
          <v-list-item class="mb-2">
            <v-list-item-title class="text-h6 font-weight-bold text-green-darken-3 mb-2">
              Latest News
            </v-list-item-title>
          </v-list-item>

          <SideNews />

          <v-divider class="my-3"></v-divider>

          <v-list-item
            @click="onLogout"
            prepend-icon="mdi-logout"
            title="Sign Out"
            class="text-red"
          ></v-list-item>
        </v-list>
      </div>
    </v-card>
  </v-dialog>
</template>

<style scoped>
.mobile-bottom-nav {
  z-index: 9999 !important;
  position: fixed !important;
  bottom: 0 !important;
}

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

.message-badge {
  font-weight: bold;
}

.mobile-menu-card {
  background: #f5f5f5;
}
</style>
