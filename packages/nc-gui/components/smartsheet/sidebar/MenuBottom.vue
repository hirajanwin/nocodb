<script lang="ts" setup>
import { ViewTypes } from 'nocodb-sdk'
import { useNuxtApp } from '#app'
import { useSmartsheetStoreOrThrow, useUIPermission, viewIcons } from '#imports'

interface Emits {
  (event: 'openModal', data: { type: ViewTypes; title?: string }): void
}

const emits = defineEmits<Emits>()

const { $e } = useNuxtApp()

const { isUIAllowed } = useUIPermission()

const { isSqlView } = useSmartsheetStoreOrThrow()

function onOpenModal(type: ViewTypes, title = '') {
  $e('c:view:create', { view: type })
  emits('openModal', { type, title })
}
</script>

<template>
  <a-menu :selected-keys="[]" class="flex flex-col">
    <div v-if="isUIAllowed('virtualViewsCreateOrEdit')">
      <h3 class="px-3 py-1 text-xs font-semibold flex items-center gap-4 text-gray-500">
        {{ $t('activity.createView') }}
      </h3>

      <a-menu-item
        key="grid"
        class="group !flex !items-center !my-0 !h-[30px] nc-create-grid-view"
        @click="onOpenModal(ViewTypes.GRID)"
      >
        <a-tooltip :mouse-enter-delay="1" placement="left">
          <template #title>
            {{ $t('msg.info.addView.grid') }}
          </template>

          <div class="nc-project-menu-item text-xs flex items-center h-full w-full gap-2">
            <component :is="viewIcons[ViewTypes.GRID].icon" :style="{ color: viewIcons[ViewTypes.GRID].color }" />

            <div>{{ $t('objects.viewType.grid') }}</div>

            <div class="flex-1" />

            <mdi-plus class="group-hover:text-primary" />
          </div>
        </a-tooltip>
      </a-menu-item>

      <a-menu-item
        key="gallery"
        class="group !flex !items-center !my-0 nc-create-gallery-view"
        @click="onOpenModal(ViewTypes.GALLERY)"
      >
        <a-tooltip :mouse-enter-delay="1" placement="left">
          <template #title>
            {{ $t('msg.info.addView.gallery') }}
          </template>

          <div class="nc-project-menu-item text-xs flex items-center h-full w-full gap-2">
            <component :is="viewIcons[ViewTypes.GALLERY].icon" :style="{ color: viewIcons[ViewTypes.GALLERY].color }" />

            <div>{{ $t('objects.viewType.gallery') }}</div>

            <div class="flex-1" />

            <mdi-plus class="group-hover:text-primary" />
          </div>
        </a-tooltip>
      </a-menu-item>

      <a-menu-item
        v-if="!isSqlView"
        key="form"
        class="group !flex !items-center !my-0 !h-[30px] nc-create-form-view"
        @click="onOpenModal(ViewTypes.FORM)"
      >
        <a-tooltip :mouse-enter-delay="1" placement="left">
          <template #title>
            {{ $t('msg.info.addView.form') }}
          </template>

          <div class="nc-project-menu-item text-xs flex items-center h-full w-full gap-2">
            <component :is="viewIcons[ViewTypes.FORM].icon" :style="{ color: viewIcons[ViewTypes.FORM].color }" />

            <div>{{ $t('objects.viewType.form') }}</div>

            <div class="flex-1" />

            <mdi-plus class="group-hover:text-primary" />
          </div>
        </a-tooltip>
      </a-menu-item>

      <div class="w-full h-4" />
    </div>
  </a-menu>
</template>
