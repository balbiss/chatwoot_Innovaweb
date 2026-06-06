<template>
  <div class="flex h-full w-full overflow-x-auto overflow-y-hidden bg-n-surface-1 p-4 space-x-4">
    <!-- Se não houver etiquetas -->
    <div v-if="kanbanColumns.length === 0" class="flex flex-col items-center justify-center w-full h-full">
      <h2 class="text-xl font-semibold mb-2">Configure o seu Kanban</h2>
      <p class="text-n-slate-11 text-center max-w-md">
        O seu Kanban é baseado nas Etiquetas do Chatwoot.
        Crie etiquetas (como "Lead", "Negociando", "Fechado") nas Configurações para que elas apareçam como colunas aqui.
      </p>
      <router-link :to="{ name: 'labels_wrapper' }">
        <button class="mt-4 px-4 py-2 bg-n-solid-blue text-white rounded-md hover:bg-n-solid-blue-hover transition">
          Ir para Etiquetas
        </button>
      </router-link>
    </div>

    <!-- Colunas do Kanban -->
    <div
      v-for="column in kanbanColumns"
      :key="column.id"
      class="flex flex-col flex-shrink-0 w-80 bg-n-surface-2 border border-n-weak rounded-xl p-3 shadow-sm h-full"
      @dragover.prevent
      @dragenter.prevent
      @drop="onDrop($event, column)"
    >
      <div class="flex items-center justify-between mb-4 px-1">
        <div class="flex items-center gap-2">
          <span
            class="w-3 h-3 rounded-full"
            :style="{ backgroundColor: column.color }"
          ></span>
          <h3 class="font-semibold text-n-slate-12">{{ column.title }}</h3>
        </div>
        <span class="text-xs font-medium text-n-slate-11 bg-n-surface-1 px-2 py-1 rounded-full border border-n-weak">
          {{ conversationsByLabel[column.title]?.length || 0 }}
        </span>
      </div>

      <!-- Área rolável de Cards -->
      <div class="flex-1 overflow-y-auto space-y-3 pb-2 custom-scrollbar">
        <KanbanCard
          v-for="conversation in conversationsByLabel[column.title]"
          :key="conversation.id"
          :conversation="conversation"
          draggable="true"
          @dragstart="onDragStart($event, conversation, column)"
        />
        
        <div 
          v-if="!conversationsByLabel[column.title]?.length"
          class="flex items-center justify-center h-20 border-2 border-dashed border-n-weak rounded-lg text-n-slate-10 text-sm"
        >
          Solte os cards aqui
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { mapGetters } from 'vuex';
import KanbanCard from './KanbanCard.vue';

export default {
  name: 'KanbanBoard',
  components: {
    KanbanCard,
  },
  data() {
    return {
      isLoading: false,
    };
  },
  computed: {
    ...mapGetters({
      labels: 'labels/getLabels',
      allConversations: 'getAllConversations',
      currentAccountId: 'getCurrentAccountId',
    }),
    kanbanColumns() {
      // Usar as etiquetas como colunas
      return this.labels || [];
    },
    conversationsByLabel() {
      // Agrupa as conversas abertas pela etiqueta
      const grouped = {};
      this.kanbanColumns.forEach(col => {
        grouped[col.title] = [];
      });

      // Pega apenas as conversas que estão ativas e não resolvidas
      const activeConversations = this.allConversations.filter(c => c.status !== 'resolved');

      activeConversations.forEach(conv => {
        // Se a conversa tiver labels, tentamos colocar na coluna correspondente
        if (conv.labels && conv.labels.length > 0) {
          // Pode ter várias labels, vamos colocar no primeiro match ou em todos?
          // Para um Kanban, idealmente cada conversa está em apenas uma fase (etiqueta principal)
          // Colocamos na primeira etiqueta encontrada que corresponda a uma coluna
          const mainLabel = conv.labels.find(l => grouped[l] !== undefined);
          if (mainLabel) {
            grouped[mainLabel].push(conv);
          }
        }
      });

      return grouped;
    },
  },
  mounted() {
    this.fetchData();
  },
  methods: {
    async fetchData() {
      this.isLoading = true;
      try {
        await this.$store.dispatch('labels/get');
        // Buscar conversas abertas
        await this.$store.dispatch('fetchAllConversations', { status: 'open' });
      } catch (error) {
        console.error('Erro ao buscar dados do Kanban:', error);
      } finally {
        this.isLoading = false;
      }
    },
    onDragStart(event, conversation, sourceColumn) {
      event.dataTransfer.setData('conversationId', conversation.id);
      event.dataTransfer.setData('sourceLabel', sourceColumn.title);
      event.dataTransfer.effectAllowed = 'move';
      // Add um estilo na hora do drag
      event.target.style.opacity = '0.5';
      
      // Cleanup de estilo no fim do drag
      event.target.addEventListener('dragend', function() {
        this.style.opacity = '1';
      }, { once: true });
    },
    async onDrop(event, targetColumn) {
      const conversationId = event.dataTransfer.getData('conversationId');
      const sourceLabel = event.dataTransfer.getData('sourceLabel');
      const targetLabel = targetColumn.title;

      if (!conversationId || sourceLabel === targetLabel) return;

      const conversation = this.allConversations.find(c => c.id === Number(conversationId));
      if (!conversation) return;

      // Monta a nova lista de etiquetas: remove a velha e adiciona a nova
      let updatedLabels = [...(conversation.labels || [])];
      updatedLabels = updatedLabels.filter(l => l !== sourceLabel);
      if (!updatedLabels.includes(targetLabel)) {
        updatedLabels.push(targetLabel);
      }

      // Atualização Otimista UI
      this.$store.dispatch('conversationLabels/setConversationLabel', {
        id: conversation.id,
        data: updatedLabels
      });

      try {
        await this.$store.dispatch('conversationLabels/update', {
          conversationId: conversation.id,
          labels: updatedLabels,
        });
      } catch (error) {
        // Se falhar, recarrega a página ou desfaz
        console.error('Falha ao atualizar etiqueta:', error);
      }
    },
  },
};
</script>

<style scoped>
.custom-scrollbar::-webkit-scrollbar {
  width: 4px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: transparent;
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  background-color: var(--color-border-light);
  border-radius: 4px;
}
</style>
