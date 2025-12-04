<script setup>
import { ref, computed, onMounted, watch, onUnmounted } from 'vue'
import { supabase } from '@/utils/supabase'

const props = defineProps({
  modelValue: Boolean,
  postId: Number,
  postOwnerId: String,
  postOwnerName: String,
  postOwnerAvatar: String,
  postTitle: String
})

const emit = defineEmits(['update:modelValue'])

// URL of the image
const profileUrl = 'https://ndmbunubneumkuadlylz.supabase.co/storage/v1/object/public/images/'

const message = ref('')
const messages = ref([])
const currentUserId = ref(null)
const sending = ref(false)
const loading = ref(false)
const showDeleteDialog = ref(false)
const deleting = ref(false)
let realtimeChannel = null

const isOpen = computed({
  get: () => props.modelValue,
  set: (value) => emit('update:modelValue', value)
})

// Fetch current user
const fetchCurrentUser = async () => {
  const { data: { user } } = await supabase.auth.getUser()
  if (user) {
    currentUserId.value = user.id
  }
}

// Fetch messages for this conversation
const fetchMessages = async () => {
  if (!props.postId || !currentUserId.value) return
  
  loading.value = true
  
  const { data, error } = await supabase
    .from('messages')
    .select(`
      id,
      message,
      sender_id,
      receiver_id,
      is_read,
      created_at
    `)
    .eq('post_id', props.postId)
    .or(`and(sender_id.eq.${currentUserId.value},receiver_id.eq.${props.postOwnerId}),and(sender_id.eq.${props.postOwnerId},receiver_id.eq.${currentUserId.value})`)
    .order('created_at', { ascending: true })

  if (error) {
    console.error('Error fetching messages:', error)
  } else {
    messages.value = data || []
    
    // Mark messages as read
    markMessagesAsRead()
  }
  
  loading.value = false
}

// Mark received messages as read
const markMessagesAsRead = async () => {
  if (!currentUserId.value) return
  
  const unreadMessages = messages.value.filter(
    m => m.receiver_id === currentUserId.value && !m.is_read
  )
  
  if (unreadMessages.length === 0) return
  
  await supabase
    .from('messages')
    .update({ is_read: true })
    .in('id', unreadMessages.map(m => m.id))
  
  // Notify other components to refresh unread count
  window.dispatchEvent(new CustomEvent('messages-read'));
}

// Send message
const sendMessage = async () => {
  if (!message.value.trim() || sending.value) return
  
  sending.value = true
  
  console.log('ChatModal: Sending message...')
  
  const { data, error } = await supabase
    .from('messages')
    .insert([{
      sender_id: currentUserId.value,
      receiver_id: props.postOwnerId,
      post_id: props.postId,
      message: message.value.trim()
    }])
    .select()
  
  if (error) {
    console.error('ChatModal: Error sending message:', error)
  } else {
    console.log('ChatModal: Message sent successfully:', data[0])
    messages.value.push(data[0])
    message.value = ''
    // Scroll to bottom
    setTimeout(scrollToBottom, 100)
  }
  
  sending.value = false
}

// Delete conversation
const deleteConversation = async () => {
  if (!props.postId || !currentUserId.value) return
  
  deleting.value = true
  
  try {
    // Simple approach: Permanently delete all messages in this conversation for BOTH users
    const { error: deleteError } = await supabase
      .from('messages')
      .delete()
      .eq('post_id', props.postId)
      .or(`and(sender_id.eq.${currentUserId.value},receiver_id.eq.${props.postOwnerId}),and(sender_id.eq.${props.postOwnerId},receiver_id.eq.${currentUserId.value})`)
    
    if (deleteError) {
      console.error('Error deleting messages:', deleteError)
      alert('Failed to delete conversation')
      deleting.value = false
      return
    }
    
    console.log('Successfully deleted conversation for both users')
    
    // Clear local messages and close modal
    messages.value = []
    showDeleteDialog.value = false
    isOpen.value = false
    
    // Notify other components to refresh
    window.dispatchEvent(new CustomEvent('conversation-deleted'))
    
  } catch (err) {
    console.error('Unexpected error:', err)
    alert('Failed to delete conversation')
  } finally {
    deleting.value = false
  }
}

// Scroll to bottom of messages
const scrollToBottom = () => {
  const container = document.querySelector('.messages-container')
  if (container) {
    container.scrollTop = container.scrollHeight
  }
}

// Format time
const formatTime = (dateString) => {
  const date = new Date(dateString)
  const now = new Date()
  const diffInHours = (now - date) / (1000 * 60 * 60)
  
  if (diffInHours < 24) {
    return date.toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit' })
  } else {
    return date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' })
  }
}

// Setup realtime subscription
const setupRealtimeSubscription = () => {
  if (!props.postId || !currentUserId.value) {
    console.log('ChatModal: Cannot setup subscription - missing postId or userId')
    return
  }

  // Unsubscribe from previous channel if exists
  if (realtimeChannel) {
    supabase.removeChannel(realtimeChannel)
  }

  console.log('ChatModal: Setting up realtime subscription for post:', props.postId)

  // Create channel for this conversation
  realtimeChannel = supabase
    .channel(`messages:${props.postId}:${currentUserId.value}`)
    .on(
      'postgres_changes',
      {
        event: 'INSERT',
        schema: 'public',
        table: 'messages',
        filter: `post_id=eq.${props.postId}`
      },
      (payload) => {
        console.log('ChatModal: Received new message via Realtime:', payload)
        // Only add message if it's part of this conversation
        const newMsg = payload.new
        if (
          (newMsg.sender_id === currentUserId.value && newMsg.receiver_id === props.postOwnerId) ||
          (newMsg.sender_id === props.postOwnerId && newMsg.receiver_id === currentUserId.value)
        ) {
          // Check if message already exists (to avoid duplicates from our own sends)
          const exists = messages.value.some(m => m.id === newMsg.id)
          if (!exists) {
            console.log('ChatModal: Adding new message to list')
            messages.value.push(newMsg)
            setTimeout(scrollToBottom, 100)
            
            // Mark as read if we're the receiver
            if (newMsg.receiver_id === currentUserId.value) {
              markMessagesAsRead()
            }
          } else {
            console.log('ChatModal: Message already exists, skipping')
          }
        }
      }
    )
    .on(
      'postgres_changes',
      {
        event: 'UPDATE',
        schema: 'public',
        table: 'messages',
        filter: `post_id=eq.${props.postId}`
      },
      (payload) => {
        console.log('ChatModal: Message updated via Realtime:', payload)
        const updatedMsg = payload.new
        
        // Check if message belongs to this conversation
        const isThisConversation = 
          (updatedMsg.sender_id === currentUserId.value && updatedMsg.receiver_id === props.postOwnerId) ||
          (updatedMsg.sender_id === props.postOwnerId && updatedMsg.receiver_id === currentUserId.value)
        
        if (!isThisConversation) return
        
        const index = messages.value.findIndex(m => m.id === updatedMsg.id)
        if (index !== -1) {
          // Update the message (e.g., read status changed)
          messages.value[index] = updatedMsg
        }
      }
    )
    .subscribe((status) => {
      console.log('ChatModal: Subscription status:', status)
      if (status === 'SUBSCRIBED') {
        console.log('ChatModal: Successfully subscribed to realtime messages')
      }
    })
}

// Cleanup realtime subscription
const cleanupRealtimeSubscription = () => {
  if (realtimeChannel) {
    supabase.removeChannel(realtimeChannel)
    realtimeChannel = null
  }
}

// Watch for modal open
watch(() => props.modelValue, (newVal) => {
  console.log('ChatModal: Modal visibility changed:', newVal)
  console.log('ChatModal: Props:', {
    postId: props.postId,
    postOwnerId: props.postOwnerId,
    postOwnerName: props.postOwnerName,
    postOwnerAvatar: props.postOwnerAvatar,
    postTitle: props.postTitle
  })
  if (newVal) {
    fetchMessages()
    setupRealtimeSubscription()
    setTimeout(scrollToBottom, 200)
  } else {
    cleanupRealtimeSubscription()
  }
})

// Also watch for postId changes
watch(() => props.postId, (newPostId) => {
  console.log('ChatModal: PostId changed:', newPostId)
  if (newPostId && props.modelValue) {
    cleanupRealtimeSubscription()
    fetchMessages()
    setupRealtimeSubscription()
  }
})

onMounted(() => {
  fetchCurrentUser()
})

onUnmounted(() => {
  cleanupRealtimeSubscription()
})
</script>

<template>
  <!-- Bottom-right chat popup -->
  <transition name="chat-slide">
    <div v-if="isOpen" class="chat-popup">
      <v-card class="chat-card rounded-xl elevation-8">
        <!-- Header -->
        <div class="chat-header pa-3 d-flex align-center justify-space-between">
          <div class="d-flex align-center flex-grow-1" style="min-width: 0;">
            <v-avatar size="40" class="mr-3">
              <v-img 
                v-if="postOwnerAvatar && typeof postOwnerAvatar === 'string' && postOwnerAvatar !== ''"
                :src="postOwnerAvatar.startsWith('http') ? postOwnerAvatar : profileUrl + postOwnerAvatar"
                alt="User Avatar"
                cover
              ></v-img>
              <v-icon v-else color="white" size="24">mdi-account</v-icon>
            </v-avatar>
            <div style="min-width: 0; flex: 1;">
              <h4 class="text-body-1 text-white font-weight-bold mb-0 text-truncate">{{ postOwnerName }}</h4>
              <p class="text-caption text-white text-opacity-90 mb-0 text-truncate">{{ postTitle }}</p>
            </div>
          </div>
          <div class="d-flex align-center flex-shrink-0">
            <v-btn icon="mdi-delete" variant="text" color="white" size="x-small" @click="showDeleteDialog = true" class="mr-1"></v-btn>
            <v-btn icon="mdi-minus" variant="text" color="white" size="x-small" @click="isOpen = false" class="mr-1"></v-btn>
            <v-btn icon="mdi-close" variant="text" color="white" size="x-small" @click="isOpen = false"></v-btn>
          </div>
        </div>

        <!-- Messages Container -->
        <div class="messages-container pa-3" v-if="!loading">
          <div v-if="messages.length === 0" class="text-center py-4">
            <v-icon size="48" color="grey-lighten-1">mdi-message-outline</v-icon>
            <p class="text-caption text-grey-darken-1 mt-2">No messages yet</p>
          </div>
          
          <div v-for="msg in messages" :key="msg.id" class="mb-2">
            <div 
              :class="[
                'message-bubble',
                msg.sender_id === currentUserId ? 'sent' : 'received'
              ]"
            >
              <p class="text-caption mb-0">{{ msg.message }}</p>
              <span class="text-caption time-text">{{ formatTime(msg.created_at) }}</span>
            </div>
          </div>
        </div>

        <div v-else class="pa-4 text-center">
          <v-progress-circular indeterminate color="green-darken-3" size="32"></v-progress-circular>
        </div>

        <!-- Message Input -->
        <v-divider></v-divider>
        <div class="pa-2">
          <v-text-field
            v-model="message"
            placeholder="Type a message..."
            variant="outlined"
            density="compact"
            rounded="lg"
            hide-details
            @keyup.enter="sendMessage"
            :disabled="sending"
            class="message-input"
          >
            <template v-slot:append-inner>
              <v-btn
                icon="mdi-send"
                color="green-darken-3"
                variant="flat"
                size="x-small"
                :disabled="!message.trim() || sending"
                :loading="sending"
                @click="sendMessage"
              ></v-btn>
            </template>
          </v-text-field>
        </div>
      </v-card>
      
      <!-- Delete Confirmation Dialog -->
      <v-dialog v-model="showDeleteDialog" max-width="480">
        <v-card class="rounded-xl">
          <v-card-title class="pa-6 d-flex align-center" style="background: linear-gradient(135deg, #d32f2f 0%, #c62828 100%);">
            <v-icon color="white" size="32" class="mr-3">mdi-alert-circle-outline</v-icon>
            <span class="text-h5 text-white font-weight-bold">Delete Conversation?</span>
          </v-card-title>
          <v-divider></v-divider>
          <v-card-text class="pa-6">
            <p class="text-body-1 mb-3">
              Are you sure you want to delete your conversation with <strong class="text-green-darken-3">{{ postOwnerName }}</strong> about <strong class="text-orange-darken-2">{{ postTitle }}</strong>?
            </p>
            <v-alert
              type="warning"
              variant="tonal"
              density="compact"
              class="mb-0"
            >
              <template v-slot:prepend>
                <v-icon size="20">mdi-alert</v-icon>
              </template>
              <div class="text-caption">
                <strong>Warning:</strong> This will permanently delete the conversation for BOTH users. This action cannot be undone.
              </div>
            </v-alert>
          </v-card-text>
          <v-divider></v-divider>
          <v-card-actions class="pa-4">
            <v-spacer></v-spacer>
            <v-btn
              color="grey-darken-1"
              variant="text"
              size="large"
              @click="showDeleteDialog = false"
              :disabled="deleting"
              class="px-6"
            >
              Cancel
            </v-btn>
            <v-btn
              color="red-darken-2"
              variant="flat"
              size="large"
              prepend-icon="mdi-delete"
              @click="deleteConversation"
              :loading="deleting"
              class="px-6"
            >
              Delete Conversation
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </div>
  </transition>
</template>

<style scoped>
.chat-popup {
  position: fixed;
  bottom: 0;
  right: 20px;
  z-index: 9999;
  width: 350px;
  max-height: 500px;
}

.chat-card {
  display: flex;
  flex-direction: column;
  height: 500px;
  max-height: 500px;
}

.chat-header {
  background: linear-gradient(135deg, #2e7d32 0%, #1b5e20 100%);
  border-radius: 12px 12px 0 0;
}

.messages-container {
  flex: 1;
  overflow-y: auto;
  background-color: #f5f5f5;
  min-height: 350px;
  max-height: 400px;
}

.message-bubble {
  max-width: 75%;
  padding: 8px 12px;
  border-radius: 12px;
  word-wrap: break-word;
}

.message-bubble.sent {
  background-color: #2e7d32;
  color: white;
  margin-left: auto;
  border-bottom-right-radius: 4px;
}

.message-bubble.received {
  background-color: white;
  color: #333;
  margin-right: auto;
  border-bottom-left-radius: 4px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}

.message-bubble.sent .time-text {
  color: rgba(255, 255, 255, 0.7) !important;
  font-size: 10px;
}

.message-bubble.received .time-text {
  color: rgba(0, 0, 0, 0.5) !important;
  font-size: 10px;
}

.message-input {
  font-size: 14px;
}

/* Slide animation */
.chat-slide-enter-active,
.chat-slide-leave-active {
  transition: all 0.3s ease;
}

.chat-slide-enter-from {
  transform: translateY(100%);
  opacity: 0;
}

.chat-slide-leave-to {
  transform: translateY(100%);
  opacity: 0;
}

/* Scrollbar styling */
.messages-container::-webkit-scrollbar {
  width: 6px;
}

.messages-container::-webkit-scrollbar-track {
  background: #f1f1f1;
}

.messages-container::-webkit-scrollbar-thumb {
  background: #888;
  border-radius: 3px;
}

.messages-container::-webkit-scrollbar-thumb:hover {
  background: #555;
}
</style>
