<script setup lang="ts">
import type { TabItem } from '~/composables'
import { TabMetaInj, until } from '#imports'

const { getMeta } = useMetas()

const { tables } = useProject()

const route = useRoute()

const loading = ref(true)

const activeTab = inject(
  TabMetaInj,
  computed(() => ({} as TabItem)),
)

/** wait until table list loads since meta load requires table list **/
until(tables)
  .toMatch((tables) => tables.length > 0)
  .then(() => {
    getMeta(route.params.title as string, true).finally(() => (loading.value = false))
  })
</script>

<template>
  <div v-if="loading" class="flex items-center justify-center h-full w-full">
    <a-spin size="large" />
  </div>
  <TabsSmartsheet v-else :key="route.params.title" :active-tab="activeTab" />
</template>
