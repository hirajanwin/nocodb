<script setup lang="ts">
import { onKeyDown } from '@vueuse/core'
import { useProvideAttachmentCell } from './utils'
import { useSortable } from './sort'
import Modal from './Modal.vue'
import Carousel from './Carousel.vue'
import {
  IsFormInj,
  IsGalleryInj,
  inject,
  isImage,
  nextTick,
  openLink,
  ref,
  useDropZone,
  useSmartsheetRowStoreOrThrow,
  useSmartsheetStoreOrThrow,
  watch,
} from '#imports'

interface Props {
  modelValue?: string | Record<string, any>[] | null
  rowIndex?: number
}

interface Emits {
  (event: 'update:modelValue', value: string | Record<string, any>): void
}

const { modelValue, rowIndex } = defineProps<Props>()

const emits = defineEmits<Emits>()

const isForm = inject(IsFormInj, ref(false))

const isGallery = inject(IsGalleryInj, ref(false))

const attachmentCellRef = ref<HTMLDivElement>()

const sortableRef = ref<HTMLDivElement>()

const { cellRefs } = useSmartsheetStoreOrThrow()!

const {
  column,
  modalVisible,
  attachments,
  visibleItems,
  onDrop,
  isLoading,
  open,
  FileIcon,
  selectedImage,
  isReadonly,
  storedFiles,
} = useProvideAttachmentCell(updateModelValue)

const currentCellRef = ref()

watch(
  [() => rowIndex, isForm],
  () => {
    if (!rowIndex && isForm.value && isGallery.value) {
      currentCellRef.value = attachmentCellRef.value
    } else {
      nextTick(() => {
        const nextCell = cellRefs.value.reduceRight((cell, curr) => {
          if (!cell && curr.dataset.key === `${rowIndex}${column.value!.id}`) cell = curr

          return cell
        }, undefined as HTMLTableDataCellElement | undefined)

        if (!nextCell) {
          currentCellRef.value = attachmentCellRef.value
        } else {
          currentCellRef.value = nextCell
        }
      })
    }
  },
  { immediate: true, flush: 'post' },
)

const { dragging } = useSortable(sortableRef, visibleItems, updateModelValue, isReadonly)

const { state: rowState } = useSmartsheetRowStoreOrThrow()

const { isOverDropZone } = useDropZone(currentCellRef, onDrop)

/** on new value, reparse our stored attachments */
watch(
  () => modelValue,
  (nextModel) => {
    if (nextModel) {
      try {
        attachments.value = ((typeof nextModel === 'string' ? JSON.parse(nextModel) : nextModel) || []).filter(Boolean)
      } catch (e) {
        console.error(e)
        attachments.value = []
      }
    }
  },
  { immediate: true },
)

/** updates attachments array for autosave */
function updateModelValue(data: string | Record<string, any>) {
  emits('update:modelValue', typeof data !== 'string' ? JSON.stringify(data) : data)
}

/** Close modal on escape press, disable dropzone as well */
onKeyDown('Escape', () => {
  modalVisible.value = false
  isOverDropZone.value = false
})

/** sync storedFiles state with row state */
watch(
  () => storedFiles.value.length || 0,
  () => {
    rowState.value[column.value!.title!] = storedFiles.value
  },
)

const { isSharedForm } = useSmartsheetStoreOrThrow()
</script>

<template>
  <div
    ref="attachmentCellRef"
    class="nc-attachment-cell relative flex-1 color-transition flex items-center justify-between gap-1"
  >
    <Carousel />

    <template v-if="isSharedForm || (!isReadonly && !dragging && !!currentCellRef)">
      <general-overlay
        v-model="isOverDropZone"
        inline
        :target="currentCellRef"
        class="text-white text-lg ring ring-accent bg-gray-700/75 flex items-center justify-center gap-2 backdrop-blur-xl"
      >
        <MaterialSymbolsFileCopyOutline class="text-accent" /> Drop here
      </general-overlay>
    </template>

    <div
      v-if="!isReadonly"
      :class="{ 'mx-auto px-4': !visibleItems.length }"
      class="group flex gap-1 items-center active:ring rounded border-1 p-1 hover:(bg-primary bg-opacity-10)"
      @click.stop="open"
    >
      <MdiReload v-if="isLoading" :class="{ 'animate-infinite animate-spin': isLoading }" />

      <a-tooltip v-else placement="bottom">
        <template #title> Click or drop a file into cell </template>

        <div class="flex items-center gap-2">
          <MaterialSymbolsAttachFile class="transform group-hover:(text-accent scale-120) text-gray-500 text-[10px]" />

          <div v-if="!visibleItems.length" class="group-hover:text-primary text-gray-500 text-xs">Add file(s)</div>
        </div>
      </a-tooltip>
    </div>
    <div v-else class="flex" />

    <template v-if="visibleItems.length">
      <div
        ref="sortableRef"
        :class="{ dragging }"
        class="flex justify-center items-center flex-wrap gap-2 p-1 scrollbar-thin-dull max-h-[150px] overflow-auto"
      >
        <div
          v-for="(item, i) of visibleItems"
          :key="item.url || item.title"
          :class="isImage(item.title, item.mimetype ?? item.type) ? '' : 'border-1 rounded'"
          class="nc-attachment flex items-center justify-center min-h-[50px]"
        >
          <a-tooltip placement="bottom">
            <template #title>
              <div class="text-center w-full">{{ item.title }}</div>
            </template>

            <nuxt-img
              v-if="isImage(item.title, item.mimetype ?? item.type) && (item.url || item.data)"
              quality="75"
              placeholder
              :alt="item.title || `#${i}`"
              :src="item.url || item.data"
              class="ring-1 ring-gray-300 rounded max-h-[50px] max-w-[50px]"
              @click="selectedImage = item"
            />

            <component :is="FileIcon(item.icon)" v-else-if="item.icon" @click="openLink(item.url || item.data)" />

            <IcOutlineInsertDriveFile v-else @click.stop="openLink(item.url || item.data)" />
          </a-tooltip>
        </div>
      </div>

      <div class="group flex gap-1 items-center border-1 active:ring rounded p-1 hover:(bg-primary bg-opacity-10)">
        <MdiReload v-if="isLoading" :class="{ 'animate-infinite animate-spin': isLoading }" />

        <a-tooltip v-else placement="bottom">
          <template #title> View attachments </template>

          <MdiArrowExpand
            class="select-none transform group-hover:(text-accent scale-120) text-[10px] text-gray-500"
            @click.stop="modalVisible = true"
          />
        </a-tooltip>
      </div>
    </template>

    <Modal />
  </div>
</template>

<style lang="scss">
.nc-cell {
  .nc-attachment-cell {
    .nc-attachment {
      @apply w-[50px] h-[50px] min-h-[50px] min-w-[50px];
    }

    .ghost,
    .ghost > * {
      @apply !pointer-events-none;
    }

    .dragging {
      .ant-tooltip {
        @apply !hidden;
      }
    }
  }
}
</style>
