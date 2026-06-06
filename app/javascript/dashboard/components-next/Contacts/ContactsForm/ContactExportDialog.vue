<script setup>
import { ref, computed } from 'vue';
import { useMapGetter } from 'dashboard/composables/store';
import { useRoute } from 'vue-router';
import { useI18n } from 'vue-i18n';
import filterQueryGenerator from 'dashboard/helper/filterQueryGenerator';

import Dialog from 'dashboard/components-next/dialog/Dialog.vue';
import Checkbox from 'dashboard/components-next/checkbox/Checkbox.vue';

const emit = defineEmits(['export']);

const { t } = useI18n();
const route = useRoute();

const dialogRef = ref(null);
const exportOnlyNameAndPhone = ref(false);

const segments = useMapGetter('customViews/getContactCustomViews');
const appliedFilters = useMapGetter('contacts/getAppliedContactFilters');
const uiFlags = useMapGetter('contacts/getUIFlags');
const isExportingContact = computed(() => uiFlags.value.isExporting);

const activeSegmentId = computed(() => route.params.segmentId);
const activeSegment = computed(() =>
  activeSegmentId.value
    ? segments.value.find(view => view.id === Number(activeSegmentId.value))
    : undefined
);

const exportContacts = async () => {
  let query = { payload: [] };

  if (activeSegmentId.value && activeSegment.value) {
    query = activeSegment.value.query;
  } else if (Object.keys(appliedFilters.value).length > 0) {
    query = filterQueryGenerator(appliedFilters.value);
  }

  const payload = {
    ...query,
    label: route.params.label || '',
  };

  if (exportOnlyNameAndPhone.value) {
    payload.column_names = ['name', 'phone_number'];
  }

  emit('export', payload);
};

const handleDialogConfirm = async () => {
  await exportContacts();
  dialogRef.value?.close();
};

defineExpose({ dialogRef });
</script>

<template>
  <Dialog
    ref="dialogRef"
    :title="t('CONTACTS_LAYOUT.HEADER.ACTIONS.EXPORT_CONTACT.TITLE')"
    :description="
      t('CONTACTS_LAYOUT.HEADER.ACTIONS.EXPORT_CONTACT.DESCRIPTION')
    "
    :confirm-button-label="
      t('CONTACTS_LAYOUT.HEADER.ACTIONS.EXPORT_CONTACT.CONFIRM')
    "
    :is-loading="isExportingContact"
    :disable-confirm-button="isExportingContact"
    @confirm="handleDialogConfirm"
  >
    <div class="mt-2 mb-2 flex items-center gap-2">
      <Checkbox v-model="exportOnlyNameAndPhone" />
      <span class="text-sm text-n-slate-11">
        {{ t('CONTACTS_LAYOUT.HEADER.ACTIONS.EXPORT_CONTACT.EXPORT_ONLY_NAME_PHONE') }}
      </span>
    </div>
  </Dialog>
</template>
