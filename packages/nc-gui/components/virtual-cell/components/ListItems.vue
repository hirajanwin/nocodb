<script lang="ts" setup>
import { RelationTypes, UITypes } from 'nocodb-sdk'
import type { ColumnType, LinkToAnotherRecordType } from 'nocodb-sdk'
import { Empty } from 'ant-design-vue'
import {
  ColumnInj,
  computed,
  defineAsyncComponent,
  inject,
  ref,
  useLTARStoreOrThrow,
  useSmartsheetRowStoreOrThrow,
  useVModel,
  watch,
} from '#imports'
import { IsPublicInj } from '~/context'

const props = defineProps<{ modelValue: boolean }>()

const emit = defineEmits(['update:modelValue', 'addNewRecord'])

const ExpandedForm: any = defineAsyncComponent(() => import('../../smartsheet/expanded-form/index.vue'))

const vModel = useVModel(props, 'modelValue', emit)

const column = inject(ColumnInj)

const {
  childrenExcludedList,
  loadChildrenExcludedList,
  childrenExcludedListPagination,
  relatedTablePrimaryValueProp,
  link,
  getRelatedTableRowId,
  relatedTableMeta,
  meta,
  row,
} = useLTARStoreOrThrow()

const { addLTARRef, isNew } = useSmartsheetRowStoreOrThrow()

const isPublic = inject(IsPublicInj, ref(false))

const linkRow = async (row: Record<string, any>) => {
  if (isNew.value) {
    addLTARRef(row, column?.value as ColumnType)
  } else {
    await link(row)
  }
  vModel.value = false
}

/** reload list on modal open */
watch(vModel, (nextVal, prevVal) => {
  if (nextVal && !prevVal) {
    /** reset query and limit */
    childrenExcludedListPagination.query = ''
    childrenExcludedListPagination.page = 1
    loadChildrenExcludedList()
  }
})

const expandedFormDlg = ref(false)

/** populate initial state for a new row which is parent/child of current record */
const newRowState = computed(() => {
  if (isNew.value) return {}
  const colOpt = (column?.value as ColumnType)?.colOptions as LinkToAnotherRecordType
  const colInRelatedTable: ColumnType | undefined = relatedTableMeta?.value?.columns?.find((col) => {
    if (col.uidt !== UITypes.LinkToAnotherRecord) return false
    const colOpt1 = col?.colOptions as LinkToAnotherRecordType
    if (colOpt1?.fk_related_model_id !== meta.value.id) return false

    if (colOpt.type === RelationTypes.MANY_TO_MANY && colOpt1?.type === RelationTypes.MANY_TO_MANY) {
      return (
        colOpt.fk_parent_column_id === colOpt1.fk_child_column_id && colOpt.fk_child_column_id === colOpt1.fk_parent_column_id
      )
    } else {
      return (
        colOpt.fk_parent_column_id === colOpt1.fk_parent_column_id && colOpt.fk_child_column_id === colOpt1.fk_child_column_id
      )
    }
  })
  if (!colInRelatedTable) return {}
  const relatedTableColOpt = colInRelatedTable?.colOptions as LinkToAnotherRecordType
  if (!relatedTableColOpt) return {}

  if (relatedTableColOpt.type === RelationTypes.BELONGS_TO) {
    return {
      [colInRelatedTable.title as string]: row?.value?.row,
    }
  } else {
    return {
      [colInRelatedTable.title as string]: row?.value && [row.value.row],
    }
  }
})

// if it's an existing record close the list
// after new record creation since it's already linking while creating
watch(expandedFormDlg, (nexVal) => {
  if (!nexVal && !isNew.value) vModel.value = false
})
</script>

<template>
  <a-modal
    v-model:visible="vModel"
    :footer="null"
    :title="$t('activity.linkRecord')"
    :body-style="{ padding: 0 }"
    wrap-class-name="nc-modal-link-record"
  >
    <div class="max-h-[max(calc(100vh_-_300px)_,500px)] flex flex-col py-6">
      <div class="flex mb-4 items-center gap-2 px-12">
        <a-input
          v-model:value="childrenExcludedListPagination.query"
          :placeholder="$t('placeholder.filterQuery')"
          class="max-w-[200px]"
          size="small"
        ></a-input>
        <div class="flex-1" />
        <MdiReload class="cursor-pointer text-gray-500 nc-reload" @click="loadChildrenExcludedList" />
        <!--        Add new record -->
        <a-button v-if="!isPublic" type="primary" size="small" @click="expandedFormDlg = true">
          {{ $t('activity.addNewRecord') }}
        </a-button>
      </div>
      <template v-if="childrenExcludedList?.pageInfo?.totalRows">
        <div class="flex-1 overflow-auto min-h-0 scrollbar-thin-dull px-12">
          <a-card
            v-for="(refRow, i) in childrenExcludedList?.list ?? []"
            :key="i"
            class="!my-4 cursor-pointer hover:(!bg-gray-200/50 shadow-md) group"
            @click="linkRow(refRow)"
          >
            {{ refRow[relatedTablePrimaryValueProp] }}
            <span class="hidden group-hover:(inline) text-gray-400 text-[11px] ml-1">
              ({{ $t('labels.primaryKey') }} : {{ getRelatedTableRowId(refRow) }})
            </span>
          </a-card>
        </div>
        <div class="flex justify-center mt-6">
          <a-pagination
            v-if="childrenExcludedList?.pageInfo"
            v-model:current="childrenExcludedListPagination.page"
            v-model:page-size="childrenExcludedListPagination.size"
            class="mt-2 !text-xs"
            size="small"
            :total="childrenExcludedList.pageInfo.totalRows"
            show-less-items
          />
        </div>
      </template>
      <a-empty v-else class="my-10" :image="Empty.PRESENTED_IMAGE_SIMPLE" />

      <Suspense>
        <ExpandedForm
          v-if="expandedFormDlg"
          v-model="expandedFormDlg"
          :meta="relatedTableMeta"
          :row="{ row: {}, oldRow: {}, rowMeta: { new: true } }"
          :state="newRowState"
          use-meta-fields
        />
      </Suspense>
    </div>
  </a-modal>
</template>

<style scoped>
:deep(.ant-pagination-item a) {
  line-height: 21px !important;
}
</style>
