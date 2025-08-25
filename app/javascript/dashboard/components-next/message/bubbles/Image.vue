<script setup>
import { ref, computed, watch, onBeforeUnmount } from 'vue';
import { useI18n } from 'vue-i18n';
import { useAlert } from 'dashboard/composables';
import BaseBubble from './Base.vue';
import Button from 'next/button/Button.vue';
import { useSnakeCase } from 'dashboard/composables/useTransformKeys';
import { useMessageContext } from '../provider.js';
import { downloadFile } from '@chatwoot/utils';
import GalleryView from 'dashboard/components/widgets/conversation/components/GalleryView.vue';

// --- State Setup ---
const emit = defineEmits(['error']);
const { t } = useI18n();
const { filteredCurrentChatAttachments, attachments } = useMessageContext();
const attachment = computed(() => attachments.value[0]);

const hasError = ref(false);
const showGallery = ref(false);
const isDownloading = ref(false);
const isImageReady = ref(false);

const retryCount = ref(0);
const maxRetries = 3;
const retryDelay = 2000;
let retryTimeout = null;
let lastUrl = null;

// --- Image Preloading Function ---
function preloadImage(url) {
  return new Promise((resolve, reject) => {
    if (!url) return reject();
    const img = new window.Image();
    img.onload = () => resolve();
    img.onerror = () => reject();
    img.src = url;
  });
}

// --- Resilient Retry Logic ---
async function tryLoadImage(url) {
  if (!url) {
    hasError.value = true;
    isImageReady.value = false;
    return;
  }
  lastUrl = url;
  for (retryCount.value = 0; retryCount.value <= maxRetries; retryCount.value++) {
    try {
      await preloadImage(url);
      if (url === lastUrl) { // only update if url hasn't changed
        isImageReady.value = true;
        hasError.value = false;
      }
      return;
    } catch {
      if (retryCount.value === maxRetries) {
        if (url === lastUrl) {
          hasError.value = true;
          isImageReady.value = false;
          emit('error');
        }
      } else {
        // Wait before retrying, but be interruptible if url changes
        await new Promise((resolve) => {
          retryTimeout = setTimeout(() => {
            retryTimeout = null;
            resolve();
          }, retryDelay);
        });
        if (url !== lastUrl) return; // attachment changed, exit early
      }
    }
  }
}

// --- Watch for Attachment Change ---
watch(
  () => attachment.value?.dataUrl,
  (url) => {
    // Reset state and begin loading if URL changes
    if (retryTimeout) clearTimeout(retryTimeout);
    isImageReady.value = false;
    hasError.value = false;
    retryCount.value = 0;
    if (url) tryLoadImage(url);
  },
  { immediate: true }
);

onBeforeUnmount(() => {
  if (retryTimeout) clearTimeout(retryTimeout);
});

// --- Download Logic ---
const downloadAttachment = async () => {
  const { fileType, dataUrl, extension } = attachment.value;
  try {
    isDownloading.value = true;
    await downloadFile({ url: dataUrl, type: fileType, extension });
  } catch {
    useAlert(t('GALLERY_VIEW.ERROR_DOWNLOADING'));
  } finally {
    isDownloading.value = false;
  }
};
</script>

<template>
  <BaseBubble
    class="overflow-hidden p-3"
    data-bubble-name="image"
    @click="() => { showGallery = true; }"
  >
    <!-- Only render the image if it is ready -->
    <div v-if="isImageReady" class="relative group rounded-lg overflow-hidden">
      <img
        class="skip-context-menu"
        :src="attachment.dataUrl"
        :width="attachment.width"
        :height="attachment.height"
        draggable="false"
      />
      <div
        class="inset-0 p-2 pointer-events-none absolute bg-gradient-to-tl from-n-slate-12/30 dark:from-n-slate-1/50 via-transparent to-transparent hidden group-hover:flex"
      />
      <div class="absolute right-2 bottom-2 hidden group-hover:flex gap-2">
        <Button xs solid slate icon="i-lucide-expand" class="opacity-60" />
        <Button
          xs
          solid
          slate
          icon="i-lucide-download"
          class="opacity-60"
          :is-loading="isDownloading"
          :disabled="isDownloading"
          @click.stop="downloadAttachment"
        />
      </div>
    </div>
    <!-- Nothing shown if not ready or errored -->
  </BaseBubble>
  <GalleryView
    v-if="showGallery"
    v-model:show="showGallery"
    :attachment="useSnakeCase(attachment)"
    :all-attachments="filteredCurrentChatAttachments"
    @close="() => (showGallery = false)"
  />
</template>