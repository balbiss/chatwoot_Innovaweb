<template>
  <div class="bg-n-solid-1 p-3 rounded-lg border border-n-weak shadow-sm cursor-grab hover:border-n-slate-6 transition-colors">
    <div class="flex items-center gap-3 mb-2">
      <Avatar
        :src="contact.thumbnail"
        :name="contact.name"
        :status="contact.availability_status"
        :size="32"
      />
      <div class="flex flex-col overflow-hidden">
        <span class="font-medium text-sm text-n-slate-12 truncate">{{ contact.name || 'Desconhecido' }}</span>
        <span class="text-xs text-n-slate-11 truncate">{{ conversation.inbox_id ? inboxName : 'Caixa de Entrada' }}</span>
      </div>
    </div>
    
    <div class="text-xs text-n-slate-11 mt-2 line-clamp-2">
      {{ lastMessageContent || 'Nenhuma mensagem' }}
    </div>

    <!-- Rodapé do Card: Data e Avatar do Atendente (Assignee) -->
    <div class="flex items-center justify-between mt-3 pt-2 border-t border-n-weak">
      <span class="text-[10px] text-n-slate-10">{{ formattedDate }}</span>
      <Avatar
        v-if="assignee"
        :src="assignee.thumbnail"
        :name="assignee.name"
        :size="20"
        class="border border-n-weak rounded-full"
      />
    </div>
  </div>
</template>

<script>
import Avatar from 'dashboard/components-next/avatar/Avatar.vue';
import { mapGetters } from 'vuex';

export default {
  name: 'KanbanCard',
  components: {
    Avatar,
  },
  props: {
    conversation: {
      type: Object,
      required: true,
    },
  },
  computed: {
    ...mapGetters({
      inboxes: 'inboxes/getInboxes',
    }),
    contact() {
      return this.conversation.meta?.sender || {};
    },
    assignee() {
      return this.conversation.meta?.assignee;
    },
    lastMessageContent() {
      const messages = this.conversation.messages || [];
      const validMessages = messages.filter(m => m.message_type === 0 || m.message_type === 1);
      if (validMessages.length > 0) {
        return validMessages[validMessages.length - 1].content;
      }
      return '';
    },
    inboxName() {
      const inbox = this.inboxes.find(i => i.id === this.conversation.inbox_id);
      return inbox ? inbox.name : '';
    },
    formattedDate() {
      if (!this.conversation.timestamp) return '';
      const date = new Date(this.conversation.timestamp * 1000);
      return date.toLocaleDateString() + ' ' + date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
    }
  },
};
</script>
