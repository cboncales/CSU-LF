<script setup>
import { ref, computed, onMounted, watch, onUnmounted } from 'vue'
import { supabase } from '@/utils/supabase'

const conversations = ref([])
const currentUserId = ref(null)
const selectedConversation = ref(null)
const messages = ref([])
const newMessage = ref('')
const loading = ref(false)
const sending = ref(false)
const searchQuery = ref('')
const showDeleteDialog = ref(false)
const conversationToDelete = ref(null)
const deleting = ref(false)
let realtimeChannel = null

const profileUrl = 'https://ndmbunubneumkuadlylz.supabase.co/storage/v1/object/public/images/'

// Fetch current user
const fetchCurrentUser = async () => {
  const { data: { user } } = await supabase.auth.getUser()
  if (user) {
    currentUserId.value = user.id
  }
}

// Fetch all conversations
const fetchConversations = async () => {
  if (!currentUserId.value) return
  
  loading.value = true
  
  // First, fetch all posts with user info to get user details
  const { data: allPosts } = await supabase.rpc('get_posts_with_user_info')
  
  // Create a map of userId to user info
  const userMap = new Map()
  if (allPosts) {
    for (const post of allPosts) {
      if (!userMap.has(post.user_id)) {
        userMap.set(post.user_id, {
          firstname: post.firstname,
          lastname: post.lastname,
          full_name: post.full_name,
          profile_pic: post.profile_pic,
          avatar_url: post.avatar_url
        })
      }
    }
  }
  
  const { data, error } = await supabase
    .from('messages')
    .select(`
      id,
      sender_id,
      receiver_id,
      post_id,
      message,
      is_read,
      created_at
    `)
    .or(`sender_id.eq.${currentUserId.value},receiver_id.eq.${currentUserId.value}`)
    .order('created_at', { ascending: false })

  if (error) {
    console.error('Error fetching conversations:', error)
    loading.value = false
    return
  }

  // Group messages by conversation partner and post
  const convMap = new Map()
  
  for (const msg of data || []) {
    const partnerId = msg.sender_id === currentUserId.value ? msg.receiver_id : msg.sender_id
    const key = `${partnerId}-${msg.post_id}`
    
    if (!convMap.has(key)) {
      // Get partner's user info from userMap
      const partnerInfo = userMap.get(partnerId)
      
      // Fetch post info
      const postInfo = allPosts?.find(p => p.post_id === msg.post_id)
      
      // Get partner's name and avatar
      const partnerName = partnerInfo?.firstname && partnerInfo?.lastname
        ? `${partnerInfo.firstname} ${partnerInfo.lastname}`
        : partnerInfo?.full_name || 'Unknown User'
      const partnerAvatar = partnerInfo?.profile_pic || partnerInfo?.avatar_url
      
      convMap.set(key, {
        partnerId,
        postId: msg.post_id,
        partnerName,
        partnerAvatar,
        postTitle: postInfo?.item_name || 'Unknown Item',
        lastMessage: msg.message,
        lastMessageTime: msg.created_at,
        unreadCount: 0
      })
    }
    
    // Count unread messages
    if (msg.receiver_id === currentUserId.value && !msg.is_read) {
      const conv = convMap.get(key)
      conv.unreadCount++
    }
  }
  
  conversations.value = Array.from(convMap.values())
  loading.value = false
}

// Fetch messages for selected conversation
const fetchMessages = async () => {
  if (!selectedConversation.value || !currentUserId.value) return
  
  const { data, error } = await supabase
    .from('messages')
    .select('*')
    .eq('post_id', selectedConversation.value.postId)
    .or(`and(sender_id.eq.${currentUserId.value},receiver_id.eq.${selectedConversation.value.partnerId}),and(sender_id.eq.${selectedConversation.value.partnerId},receiver_id.eq.${currentUserId.value})`)
    .order('created_at', { ascending: true })

  if (error) {
    console.error('Error fetching messages:', error)
  } else {
    messages.value = data || []
    
    // Mark as read
    markMessagesAsRead()
  }
}

// Mark messages as read
const markMessagesAsRead = async () => {
  if (!currentUserId.value || !selectedConversation.value) return
  
  const unreadMessages = messages.value.filter(
    m => m.receiver_id === currentUserId.value && !m.is_read
  )
  
  if (unreadMessages.length === 0) return
  
  await supabase
    .from('messages')
    .update({ is_read: true })
    .in('id', unreadMessages.map(m => m.id))
  
  // Update conversation unread count
  selectedConversation.value.unreadCount = 0
  
  // Notify other components to refresh unread count
  window.dispatchEvent(new CustomEvent('messages-read'));
}

// Send message
const sendMessage = async () => {
  if (!newMessage.value.trim() || sending.value || !selectedConversation.value) return
  
  sending.value = true
  
  const { data, error } = await supabase
    .from('messages')
    .insert([{
      sender_id: currentUserId.value,
      receiver_id: selectedConversation.value.partnerId,
      post_id: selectedConversation.value.postId,
      message: newMessage.value.trim()
    }])
    .select()
  
  if (error) {
    console.error('Error sending message:', error)
  } else {
    messages.value.push(data[0])
    newMessage.value = ''
    // Update last message in conversation
    selectedConversation.value.lastMessage = data[0].message
    selectedConversation.value.lastMessageTime = data[0].created_at
    setTimeout(scrollToBottom, 100)
  }
  
  sending.value = false
}

// Select conversation
const selectConversation = (conv) => {
  selectedConversation.value = conv
  fetchMessages()
}

// Open delete dialog
const confirmDeleteConversation = (conv, event) => {
  if (event) event.stopPropagation()
  conversationToDelete.value = conv
  showDeleteDialog.value = true
}

// Delete conversation
const deleteConversation = async () => {
  if (!conversationToDelete.value || !currentUserId.value) return
  
  deleting.value = true
  
  try {
    const conv = conversationToDelete.value
    
    // Simple approach: Permanently delete all messages in this conversation for BOTH users
    const { error: deleteError } = await supabase
      .from('messages')
      .delete()
      .eq('post_id', conv.postId)
      .or(`and(sender_id.eq.${currentUserId.value},receiver_id.eq.${conv.partnerId}),and(sender_id.eq.${conv.partnerId},receiver_id.eq.${currentUserId.value})`)
    
    if (deleteError) {
      console.error('Error deleting messages:', deleteError)
      alert('Failed to delete conversation')
      deleting.value = false
      return
    }
    
    console.log('Successfully deleted conversation for both users')
    
    // Remove from conversations list
    conversations.value = conversations.value.filter(
      c => !(c.partnerId === conv.partnerId && c.postId === conv.postId)
    )
    
    // If viewing this conversation, clear messages and go back to list
    if (selectedConversation.value?.partnerId === conv.partnerId && 
        selectedConversation.value?.postId === conv.postId) {
      messages.value = [] // Clear cached messages
      selectedConversation.value = null
    }
    
    showDeleteDialog.value = false
    conversationToDelete.value = null
    
    // Notify other components
    window.dispatchEvent(new CustomEvent('conversation-deleted'))
    
  } catch (err) {
    console.error('Unexpected error:', err)
    alert('Failed to delete conversation')
  } finally {
    deleting.value = false
  }
}

// Scroll to bottom
const scrollToBottom = () => {
  const container = document.querySelector('.messages-scroll-container')
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
  } else if (diffInHours < 168) {
    return date.toLocaleDateString('en-US', { weekday: 'short' })
  } else {
    return date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' })
  }
}

// Setup realtime subscription
const setupRealtimeSubscription = () => {
  if (!currentUserId.value) {
    console.log('MessagesLayout: Cannot setup subscription - missing userId')
    return
  }

  // Unsubscribe from previous channel if exists
  if (realtimeChannel) {
    supabase.removeChannel(realtimeChannel)
  }

  console.log('MessagesLayout: Setting up realtime subscription for user:', currentUserId.value)

  // Create channel for all user's messages
  realtimeChannel = supabase
    .channel(`messages:user:${currentUserId.value}`)
    .on(
      'postgres_changes',
      {
        event: 'INSERT',
        schema: 'public',
        table: 'messages',
        filter: `sender_id=eq.${currentUserId.value}`
      },
      (payload) => {
        console.log('MessagesLayout: Received new message (sender) via Realtime:', payload)
        handleNewMessage(payload.new)
      }
    )
    .on(
      'postgres_changes',
      {
        event: 'INSERT',
        schema: 'public',
        table: 'messages',
        filter: `receiver_id=eq.${currentUserId.value}`
      },
      (payload) => {
        console.log('MessagesLayout: Received new message (receiver) via Realtime:', payload)
        handleNewMessage(payload.new)
      }
    )
    .on(
      'postgres_changes',
      {
        event: 'UPDATE',
        schema: 'public',
        table: 'messages'
      },
      (payload) => {
        console.log('MessagesLayout: Message updated via Realtime:', payload)
        handleMessageUpdate(payload.new)
      }
    )
    .on(
      'postgres_changes',
      {
        event: 'DELETE',
        schema: 'public',
        table: 'messages'
      },
      (payload) => {
        console.log('MessagesLayout: Message deleted via Realtime:', payload)
        handleMessageDelete(payload.old)
      }
    )
    .subscribe((status) => {
      console.log('MessagesLayout: Subscription status:', status)
      if (status === 'SUBSCRIBED') {
        console.log('MessagesLayout: Successfully subscribed to realtime messages')
      }
    })
}

// Handle new message
const handleNewMessage = async (newMsg) => {
  // If viewing conversation list, update conversations
  if (!selectedConversation.value) {
    await fetchConversations()
  } else {
    // If in a chat, check if message belongs to current conversation
    const isCurrentConv = 
      newMsg.post_id === selectedConversation.value.postId &&
      ((newMsg.sender_id === currentUserId.value && newMsg.receiver_id === selectedConversation.value.partnerId) ||
       (newMsg.sender_id === selectedConversation.value.partnerId && newMsg.receiver_id === currentUserId.value))
    
    if (isCurrentConv) {
      // Check if message already exists
      const exists = messages.value.some(m => m.id === newMsg.id)
      if (!exists) {
        messages.value.push(newMsg)
        setTimeout(scrollToBottom, 100)
        
        // Mark as read if we're the receiver
        if (newMsg.receiver_id === currentUserId.value) {
          await supabase
            .from('messages')
            .update({ is_read: true })
            .eq('id', newMsg.id)
          
          // Notify other components to refresh unread count
          window.dispatchEvent(new CustomEvent('messages-read'));
        }
      }
    } else {
      // Message for different conversation, refresh list
      await fetchConversations()
    }
  }
}

// Handle message update (e.g., read status, soft delete)
const handleMessageUpdate = (updatedMsg) => {
  console.log('handleMessageUpdate called:', {
    messageId: updatedMsg.id,
    sender: updatedMsg.sender_id,
    receiver: updatedMsg.receiver_id,
    deleted_by_sender: updatedMsg.deleted_by_sender,
    deleted_by_receiver: updatedMsg.deleted_by_receiver,
    currentUser: currentUserId.value
  })
  
  // If viewing a specific conversation, update the messages array
  if (selectedConversation.value) {
    const index = messages.value.findIndex(m => m.id === updatedMsg.id)
    if (index !== -1) {
      // Check if current user has deleted this message
      const isSender = updatedMsg.sender_id === currentUserId.value
      const isReceiver = updatedMsg.receiver_id === currentUserId.value
      const isDeletedByMe = (isSender && updatedMsg.deleted_by_sender) || (isReceiver && updatedMsg.deleted_by_receiver)
      
      console.log('Message in current conversation:', {
        isSender,
        isReceiver,
        isDeletedByMe
      })
      
      if (isDeletedByMe) {
        // Remove from messages array (it's been soft deleted by current user)
        console.log('Removing message from array (deleted by me)')
        messages.value.splice(index, 1)
      } else {
        // Update the message (e.g., read status changed)
        console.log('Updating message in array')
        messages.value[index] = updatedMsg
      }
    }
  }
  
  // Check if other user deleted a message - refresh conversations
  // to remove conversations where all messages are deleted
  const isSender = updatedMsg.sender_id === currentUserId.value
  const isReceiver = updatedMsg.receiver_id === currentUserId.value
  const otherUserDeleted = (isSender && updatedMsg.deleted_by_receiver) || (isReceiver && updatedMsg.deleted_by_sender)
  
  console.log('Checking if other user deleted:', {
    isSender,
    isReceiver,
    otherUserDeleted
  })
  
  if (otherUserDeleted) {
    // Other user deleted a message, refresh conversations list
    console.log('Other user deleted - refreshing conversations')
    fetchConversations()
  }
}

// Handle message deletion (permanent delete when both users deleted)
const handleMessageDelete = (deletedMsg) => {
  console.log('handleMessageDelete called:', deletedMsg)
  
  // Remove from current conversation messages if viewing
  if (selectedConversation.value) {
    const index = messages.value.findIndex(m => m.id === deletedMsg.id)
    if (index !== -1) {
      console.log('Removing permanently deleted message from array')
      messages.value.splice(index, 1)
    }
  }
  
  // Refresh conversations list (conversation may be empty now)
  console.log('Message permanently deleted - refreshing conversations')
  fetchConversations()
}

// Cleanup realtime subscription
const cleanupRealtimeSubscription = () => {
  if (realtimeChannel) {
    supabase.removeChannel(realtimeChannel)
    realtimeChannel = null
  }
}

// Filtered conversations
const filteredConversations = computed(() => {
  if (!searchQuery.value) return conversations.value
  
  const query = searchQuery.value.toLowerCase()
  return conversations.value.filter(conv => 
    conv.partnerName.toLowerCase().includes(query) ||
    conv.postTitle.toLowerCase().includes(query)
  )
})

// Watch for selected conversation changes
watch(selectedConversation, () => {
  if (selectedConversation.value) {
    setTimeout(scrollToBottom, 200)
  }
})

onMounted(async () => {
  await fetchCurrentUser()
  await fetchConversations()
  setupRealtimeSubscription()
  
  // Listen for conversation deletions from other components
  window.addEventListener('conversation-deleted', fetchConversations)
})

onUnmounted(() => {
  cleanupRealtimeSubscription()
  window.removeEventListener('conversation-deleted', fetchConversations)
})
</script>

<template>
  <v-container fluid class="pa-4">
    <!-- Conversations List View -->
    <v-card v-if="!selectedConversation" class="rounded-xl" elevation="3">
      <!-- Header -->
      <div class="conversations-header pa-6">
        <h2 class="text-h4 font-weight-bold text-white mb-4">Messages</h2>
        <v-text-field
          v-model="searchQuery"
          placeholder="Search conversations..."
          variant="solo"
          density="comfortable"
          rounded="lg"
          hide-details
          prepend-inner-icon="mdi-magnify"
          bg-color="white"
          flat
        />
      </div>

      <!-- Conversations Grid -->
      <div class="pa-4">
        <div v-if="loading" class="text-center py-12">
          <v-progress-circular indeterminate color="green-darken-3" size="64"></v-progress-circular>
          <p class="text-body-1 text-grey-darken-1 mt-4">Loading conversations...</p>
        </div>

        <div v-else-if="filteredConversations.length === 0" class="text-center py-12">
          <v-icon size="120" color="grey-lighten-2">mdi-message-outline</v-icon>
          <h3 class="text-h6 text-grey-darken-1 mt-4">No conversations yet</h3>
          <p class="text-body-2 text-grey">Start messaging by clicking "Contact" on any post</p>
        </div>

        <v-row v-else dense>
          <v-col v-for="conv in filteredConversations" :key="`${conv.partnerId}-${conv.postId}`" cols="12">
            <v-card
              @click="selectConversation(conv)"
              class="conversation-card rounded-xl pa-4"
              elevation="2"
              hover
            >
              <div class="d-flex align-center">
                <v-badge
                  :content="conv.unreadCount"
                  :model-value="conv.unreadCount > 0"
                  color="red"
                  overlap
                  offset-x="10"
                  offset-y="10"
                >
                  <v-avatar size="64" class="mr-4 elevation-2">
                    <v-img
                      :src="conv.partnerAvatar?.startsWith('http') ? conv.partnerAvatar : profileUrl + conv.partnerAvatar"
                      alt="User Avatar"
                      cover
                    />
                  </v-avatar>
                </v-badge>

                <div class="flex-grow-1">
                  <div class="d-flex align-center justify-space-between mb-1">
                    <h3 class="text-h6 font-weight-bold text-green-darken-3">{{ conv.partnerName }}</h3>
                    <span class="text-caption text-grey">{{ formatTime(conv.lastMessageTime) }}</span>
                  </div>
                  
                  <div class="d-flex align-center mb-2">
                    <v-icon size="16" color="green-darken-2" class="mr-1">mdi-tag</v-icon>
                    <span class="text-caption text-grey-darken-1 font-weight-medium">{{ conv.postTitle }}</span>
                  </div>

                  <p class="text-body-2 text-grey-darken-2 mb-0 last-message">
                    {{ conv.lastMessage?.substring(0, 80) }}{{ conv.lastMessage?.length > 80 ? '...' : '' }}
                  </p>
                </div>

                <v-icon class="ml-3" color="grey-lighten-1">mdi-chevron-right</v-icon>
              </div>
            </v-card>
          </v-col>
        </v-row>
      </div>
    </v-card>

    <!-- Chat View -->
    <v-card v-else class="rounded-xl" elevation="3" style="height: calc(100vh - 120px);">
      <div class="d-flex flex-column h-100">
        <!-- Chat Header -->
        <div class="chat-header pa-4 d-flex align-center">
          <v-btn
            icon="mdi-arrow-left"
            variant="text"
            color="white"
            size="small"
            @click="selectedConversation = null"
            class="mr-3"
          ></v-btn>
          <v-avatar size="48" class="mr-3 elevation-2">
            <v-img
              :src="selectedConversation.partnerAvatar?.startsWith('http') ? selectedConversation.partnerAvatar : profileUrl + selectedConversation.partnerAvatar"
              alt="User Avatar"
              cover
            />
          </v-avatar>
          <div class="flex-grow-1">
            <h3 class="text-h6 font-weight-bold text-white">{{ selectedConversation.partnerName }}</h3>
            <div class="d-flex align-center">
              <v-icon size="14" color="white" class="mr-1">mdi-tag</v-icon>
              <p class="text-caption text-white text-opacity-90 mb-0">{{ selectedConversation.postTitle }}</p>
            </div>
          </div>
          <v-btn
            icon="mdi-delete"
            variant="text"
            color="red-darken-2"
            size="small"
            @click.stop="confirmDeleteConversation(selectedConversation, $event)"
          ></v-btn>
        </div>

        <!-- Messages -->
        <v-divider></v-divider>
        <div class="messages-scroll-container flex-grow-1 pa-4">
          <div v-if="messages.length === 0" class="text-center py-12">
            <v-icon size="80" color="grey-lighten-1">mdi-message-outline</v-icon>
            <p class="text-body-2 text-grey-darken-1 mt-4">No messages yet. Start the conversation!</p>
          </div>

          <div v-for="msg in messages" :key="msg.id" class="mb-3">
            <div 
              :class="[
                'message-bubble',
                msg.sender_id === currentUserId ? 'sent' : 'received'
              ]"
            >
              <p class="text-body-2 mb-1">{{ msg.message }}</p>
              <span class="text-caption">{{ formatTime(msg.created_at) }}</span>
            </div>
          </div>
        </div>

        <!-- Message Input -->
        <v-divider></v-divider>
        <div class="pa-4 message-input-container">
          <v-text-field
            v-model="newMessage"
            placeholder="Type your message..."
            variant="outlined"
            density="comfortable"
            rounded="lg"
            hide-details
            @keyup.enter="sendMessage"
            :disabled="sending"
            bg-color="grey-lighten-5"
          >
            <template v-slot:prepend-inner>
              <v-icon color="grey">mdi-emoticon-happy-outline</v-icon>
            </template>
            <template v-slot:append-inner>
              <v-btn
                icon="mdi-send"
                color="green-darken-3"
                variant="flat"
                size="small"
                :disabled="!newMessage.trim() || sending"
                :loading="sending"
                @click="sendMessage"
              ></v-btn>
            </template>
          </v-text-field>
        </div>
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
            Are you sure you want to delete your conversation with <strong class="text-green-darken-3">{{ conversationToDelete?.partnerName }}</strong> about <strong class="text-orange-darken-2">{{ conversationToDelete?.postTitle }}</strong>?
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
  </v-container>
</template>

<style scoped>
.conversations-header {
  background: linear-gradient(135deg, #2e7d32 0%, #1b5e20 100%);
  border-radius: 12px 12px 0 0;
}

.conversation-card {
  cursor: pointer;
  transition: all 0.3s ease;
  border: 2px solid transparent;
}

.conversation-card:hover {
  transform: translateY(-2px);
  border-color: #2e7d32;
  box-shadow: 0 8px 24px rgba(46, 125, 50, 0.2) !important;
}

.last-message {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.chat-header {
  background: linear-gradient(135deg, #2e7d32 0%, #1b5e20 100%);
}

.messages-scroll-container {
  overflow-y: auto;
  background: linear-gradient(to bottom, #f8f9fa, #ffffff);
}

.message-input-container {
  background-color: white;
  border-top: 1px solid #e0e0e0;
}

.message-bubble {
  max-width: 70%;
  padding: 12px 16px;
  border-radius: 16px;
  word-wrap: break-word;
  animation: slideIn 0.3s ease;
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.message-bubble.sent {
  background: linear-gradient(135deg, #2e7d32 0%, #1b5e20 100%);
  color: white;
  margin-left: auto;
  border-bottom-right-radius: 4px;
  box-shadow: 0 2px 8px rgba(46, 125, 50, 0.3);
}

.message-bubble.received {
  background-color: white;
  color: #333;
  margin-right: auto;
  border-bottom-left-radius: 4px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  border: 1px solid #e0e0e0;
}

.message-bubble.sent .text-caption {
  color: rgba(255, 255, 255, 0.8) !important;
}

.message-bubble.received .text-caption {
  color: rgba(0, 0, 0, 0.6) !important;
}

/* Scrollbar styling */
.messages-scroll-container::-webkit-scrollbar {
  width: 8px;
}

.messages-scroll-container::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 4px;
}

.messages-scroll-container::-webkit-scrollbar-thumb {
  background: #2e7d32;
  border-radius: 4px;
}

.messages-scroll-container::-webkit-scrollbar-thumb:hover {
  background: #1b5e20;
}
</style>
