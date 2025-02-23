<script setup lang="ts">
import type { ColumnType, TableType } from 'nocodb-sdk'
import SmartsheetGrid from '../smartsheet/Grid.vue'
import {
  ActiveViewInj,
  FieldsInj,
  IsFormInj,
  IsLockedInj,
  MetaInj,
  OpenNewRecordFormHookInj,
  ReloadViewDataHookInj,
  ReloadViewMetaHookInj,
  computed,
  inject,
  provide,
  provideSidebar,
  useMetas,
  useProvideSmartsheetStore,
  watch,
} from '#imports'
import type { TabItem } from '~/composables'

const { activeTab } = defineProps<{
  activeTab: TabItem
}>()

const { metas } = useMetas()

const activeView = ref()

const el = ref<typeof SmartsheetGrid>()

const fields = ref<ColumnType[]>([])

provide(TabMetaInj, ref(activeTab))
const meta = computed<TableType>(() => metas.value?.[activeTab?.id as string])

const reloadEventHook = createEventHook<void>()
const reloadViewMetaEventHook = createEventHook<void>()
const openNewRecordFormHook = createEventHook<void>()

const { isGallery, isGrid, isForm, isLocked } = useProvideSmartsheetStore(activeView, meta)

// provide the sidebar injection state
provideSidebar('nc-right-sidebar', { useStorage: true, isOpen: true })

// todo: move to store
provide(MetaInj, meta)
provide(ActiveViewInj, activeView)
provide(IsLockedInj, isLocked)
provide(ReloadViewDataHookInj, reloadEventHook)
provide(ReloadViewMetaHookInj, reloadViewMetaEventHook)
provide(OpenNewRecordFormHookInj, openNewRecordFormHook)
provide(FieldsInj, fields)
provide(IsFormInj, isForm)

const treeViewIsLockedInj = inject('TreeViewIsLockedInj', ref(false))

watch(isLocked, (nextValue) => (treeViewIsLockedInj.value = nextValue), { immediate: true })
</script>

<template>
  <div class="nc-container flex h-full">
    <div class="flex flex-col h-full flex-1 min-w-0">
      <SmartsheetToolbar />

      <template v-if="meta">
        <div class="flex flex-1 min-h-0">
          <div v-if="activeView" class="h-full flex-1 min-w-0 min-h-0 bg-gray-50">
            <SmartsheetGrid v-if="isGrid" :ref="el" />

            <SmartsheetGallery v-else-if="isGallery" />

            <SmartsheetForm v-else-if="isForm && !$route.query.reload" />
          </div>
        </div>
      </template>
    </div>
    <SmartsheetSidebar v-if="meta" class="nc-right-sidebar" />
  </div>
</template>

<style scoped>
:deep(.nc-right-sidebar.ant-layout-sider-collapsed) {
  @apply !w-0 !max-w-0 !min-w-0 overflow-x-hidden;
}
</style>
