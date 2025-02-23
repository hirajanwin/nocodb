<script lang="ts" setup>
import { ViewTypes } from 'nocodb-sdk'
import { Empty, message } from 'ant-design-vue'
import { extractSdkResponseErrorMsg, onMounted, useCopy, useI18n, useSmartsheetStoreOrThrow } from '#imports'
import MdiVisibilityOnIcon from '~icons/mdi/visibility'
import MdiVisibilityOffIcon from '~icons/mdi/visibility-off'
import MdiCopyIcon from '~icons/mdi/content-copy'
import MdiDeleteIcon from '~icons/mdi/delete-outline'

interface SharedViewType {
  password: string
  title: string
  uuid: string
  type: ViewTypes
  meta: string | Record<string, any>
  showPassword?: boolean
}

const { t } = useI18n()

const { $api, meta } = useSmartsheetStoreOrThrow()

const { copy } = useCopy()

const { dashboardUrl } = useDashboard()

const sharedViewList = ref<SharedViewType[]>()

const loadSharedViewsList = async () => {
  const list = await $api.dbViewShare.list(meta.value?.id as string)

  console.log(unref(sharedViewList))
  console.log(list)

  sharedViewList.value = list

  // todo: show active view in list separately
  // const index = sharedViewList.value.findIndex((v) => {
  //   return view?.value?.id === v.id
  // })
  //
  // if (index > -1) {
  //   activeSharedView = sharedViewList.value.splice(index, 1)[0]
  // } else {
  //   activeSharedView = null
  // }
}

onMounted(loadSharedViewsList)

const sharedViewUrl = (view: SharedViewType) => {
  let viewType
  switch (view.type) {
    case ViewTypes.FORM:
      viewType = 'form'
      break
    case ViewTypes.KANBAN:
      viewType = 'kanban'
      break
    default:
      viewType = 'view'
  }
  return `/nc/${viewType}/${view.uuid}`
}

const renderAllowCSVDownload = (view: SharedViewType) => {
  if (view.type === ViewTypes.GRID) {
    view.meta = (view.meta && typeof view.meta === 'string' ? JSON.parse(view.meta) : view.meta) as Record<string, any>
    return view.meta?.allowCSVDownload ? '✔️' : '❌'
  } else {
    return 'N/A'
  }
}

const copyLink = (view: SharedViewType) => {
  copy(`${dashboardUrl?.value as string}#${sharedViewUrl(view)}`)
  // Copied to clipboard
  message.success(t('msg.info.copiedToClipboard'))
}

const deleteLink = async (id: string) => {
  try {
    await $api.dbViewShare.delete(id)
    // Deleted shared view successfully
    message.success(t('msg.success.sharedViewDeleted'))
    await loadSharedViewsList()
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }
}
</script>

<template>
  <div class="w-full">
    <a-table
      class=""
      size="small"
      :data-source="sharedViewList"
      :pagination="{ position: ['bottomCenter'] }"
      :locale="{
        emptyText: $t('labels.noData'),
      }"
    >
      <template #emptyText>
        <a-empty :image="Empty.PRESENTED_IMAGE_SIMPLE" :description="$t('labels.noData')" />
      </template>
      <!-- View name -->
      <a-table-column key="title" :title="$t('labels.viewName')" data-index="title">
        <template #default="{ text }">
          <div class="text-xs" :title="text">
            {{ text }}
          </div>
        </template>
      </a-table-column>
      <!-- View Link -->
      <a-table-column key="title" :title="$t('labels.viewLink')" data-index="title">
        <template #default="{ record }">
          <nuxt-link :to="sharedViewUrl(record)" class="text-xs">
            {{ `${dashboardUrl}#${sharedViewUrl(record)}` }}
          </nuxt-link>
        </template>
      </a-table-column>
      <!-- Password -->
      <a-table-column key="password" :title="$t('labels.password')" data-index="title">
        <template #default="{ record }">
          <div class="flex items-center items-center gap-1">
            <template v-if="record.password">
              <span class="h-min">{{ record.showPassword ? record.password : '***************************' }}</span>
              <component
                :is="record.showPassword ? MdiVisibilityOffIcon : MdiVisibilityOnIcon"
                @click="record.showPassword = !record.showPassword"
              />
            </template>
          </div>
        </template>
      </a-table-column>
      <a-table-column key="meta" :title="$t('labels.downloadAllowed')" data-index="title">
        <template #default="{ record }">
          <template v-if="'meta' in record">
            <div class="text-center">{{ renderAllowCSVDownload(record) }}</div>
          </template>
        </template>
      </a-table-column>
      <!-- Actions -->
      <a-table-column key="id" :title="$t('labels.actions')" data-index="title">
        <template #default="{ record }">
          <div class="text-sm flex gap-2" :title="text">
            <MdiCopyIcon class="cursor-pointer" @click="copyLink(record)" />
            <MdiDeleteIcon class="cursor-pointer" @click="deleteLink(record.id)" />
          </div>
        </template>
      </a-table-column>
    </a-table>
  </div>
</template>

<style scoped>
:deep(.ant-pagination-item > a) {
  @apply leading-normal;
}
</style>
