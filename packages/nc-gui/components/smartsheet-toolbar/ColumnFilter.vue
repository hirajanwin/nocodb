<script setup lang="ts">
import type { FilterType } from 'nocodb-sdk'
import { UITypes } from 'nocodb-sdk'
import FieldListAutoCompleteDropdown from './FieldListAutoCompleteDropdown.vue'
import {
  ActiveViewInj,
  MetaInj,
  ReloadViewDataHookInj,
  comparisonOpList,
  computed,
  inject,
  ref,
  useNuxtApp,
  useViewFilters,
  watch,
} from '#imports'
import type { Filter } from '~/lib'

interface Props {
  nested?: boolean
  parentId?: string
  autoSave: boolean
  hookId?: string
  showLoading?: boolean
  modelValue?: Filter[]
  webHook?: boolean
}

const { nested = false, parentId, autoSave = true, hookId = null, modelValue, showLoading = true, webHook } = defineProps<Props>()

const emit = defineEmits(['update:filtersLength'])

const logicalOps = [
  { value: 'and', text: 'AND' },
  { value: 'or', text: 'OR' },
]

const meta = inject(MetaInj, ref())

const activeView = inject(ActiveViewInj, ref())

const reloadDataHook = inject(ReloadViewDataHookInj)!

const { $e } = useNuxtApp()

const { nestedFilters } = useSmartsheetStoreOrThrow()
const { filters, deleteFilter, saveOrUpdate, loadFilters, addFilter, addFilterGroup, sync } = useViewFilters(
  activeView,
  parentId,
  computed(() => autoSave),
  () => reloadDataHook.trigger(showLoading),
  modelValue || nestedFilters.value,
  !modelValue,
)

const localNestedFilters = ref()

const filterUpdateCondition = (filter: FilterType, i: number) => {
  saveOrUpdate(filter, i)
  $e('a:filter:update', {
    logical: filter.logical_op,
    comparison: filter.comparison_op,
  })
}

// todo : filter based on type
// const filterComparisonOp = (f) =>
//   comparisonOp.filter((op) => {
//     // if (
//     //   f &&
//     //   f.fk_column_id &&
//     //   this.columnsById[f.fk_column_id] &&
//     //   this.columnsById[f.fk_column_id].uidt === UITypes.LinkToAnotherRecord &&
//     //   this.columnsById[f.fk_column_id].uidt === UITypes.Lookup
//     // ) {
//     //   return !['notempty', 'empty', 'notnull', 'null'].includes(op.value)
//     // }
//     return true
//   })

const columns = computed(() => meta.value?.columns)

const types = computed(() => {
  if (!meta.value?.columns?.length) {
    return {}
  }

  return meta.value?.columns?.reduce((obj: any, col: any) => {
    switch (col.uidt) {
      case UITypes.Number:
      case UITypes.Decimal:
        obj[col.title] = obj[col.column_name] = 'number'
        break
      case UITypes.Checkbox:
        obj[col.title] = obj[col.column_name] = 'boolean'
        break
    }
    return obj
  }, {})
})

watch(
  () => activeView.value?.id,
  (n, o) => {
    if (n !== o && (hookId || !webHook)) loadFilters(hookId as string)
  },
  { immediate: true },
)

watch(
  () => filters.value.length,
  (length) => {
    emit('update:filtersLength', length ?? 0)
  },
)

const applyChanges = async (hookId?: string) => {
  await sync(hookId)

  if (!localNestedFilters.value?.length) return

  for (const nestedFilter of localNestedFilters.value) {
    if (nestedFilter.parentId) {
      await nestedFilter.applyChanges(hookId, true)
    }
  }
}

defineExpose({
  applyChanges,
  parentId,
})
</script>

<template>
  <div
    class="p-4 menu-filter-dropdown bg-gray-50 !border mt-4"
    :class="{ 'shadow min-w-[430px] max-w-[630px] max-h-[max(80vh,500px)] overflow-auto': !nested, 'border-1 w-full': nested }"
  >
    <div v-if="filters && filters.length" class="nc-filter-grid mb-2" @click.stop>
      <template v-for="(filter, i) in filters" :key="i">
        <template v-if="filter.status !== 'delete'">
          <template v-if="filter.is_group">
            <MdiCloseBox
              v-if="!filter.readOnly"
              :key="i"
              small
              class="nc-filter-item-remove-btn cursor-pointer text-grey"
              @click.stop="deleteFilter(filter, i)"
            />
            <span v-else :key="`${i}dummy`" />

            <div :key="`${i}nested`" class="flex">
              <a-select
                v-model:value="filter.logical_op"
                :dropdown-match-select-width="false"
                class="shrink grow-0"
                placeholder="Group op"
                dropdown-class-name="nc-dropdown-filter-logical-op-group"
                @click.stop
                @change="saveOrUpdate(filter, i)"
              >
                <a-select-option v-for="op in logicalOps" :key="op.value" :value="op.value" class="">
                  {{ op.text }}
                </a-select-option>
              </a-select>
            </div>
            <span class="col-span-3" />
            <div class="col-span-5">
              <SmartsheetToolbarColumnFilter
                v-if="filter.id || filter.children"
                ref="localNestedFilters"
                v-model="filter.children"
                :parent-id="filter.id"
                nested
                :auto-save="autoSave"
              />
            </div>
          </template>
          <template v-else>
            <!--                        <v-icon
                                      v-if="!filter.readOnly"
                                      :key="`${i}_3`"
                                      small
                                      class="nc-filter-item-remove-btn"
                                      @click.stop="deleteFilter(filter, i)"
                                    >
                                      mdi-close-box
                                    </v-icon> -->

            <MdiCloseBox
              v-if="!filter.readOnly"
              class="nc-filter-item-remove-btn text-grey self-center"
              @click.stop="deleteFilter(filter, i)"
            />
            <span v-else />

            <span v-if="!i" class="flex items-center">{{ $t('labels.where') }}</span>

            <a-select
              v-else
              v-model:value="filter.logical_op"
              :dropdown-match-select-width="false"
              class="h-full"
              hide-details
              :disabled="filter.readOnly"
              dropdown-class-name="nc-dropdown-filter-logical-op"
              @click.stop
              @change="filterUpdateCondition(filter, i)"
            >
              <a-select-option v-for="op in logicalOps" :key="op.value" :value="op.value">
                {{ op.text }}
              </a-select-option>
            </a-select>

            <FieldListAutoCompleteDropdown
              :key="`${i}_6`"
              v-model="filter.fk_column_id"
              class="nc-filter-field-select"
              :columns="columns"
              :disabled="filter.readOnly"
              @click.stop
              @change="saveOrUpdate(filter, i)"
            />

            <a-select
              v-model:value="filter.comparison_op"
              :dropdown-match-select-width="false"
              class="caption nc-filter-operation-select"
              :placeholder="$t('labels.operation')"
              density="compact"
              variant="solo"
              :disabled="filter.readOnly"
              hide-details
              dropdown-class-name="nc-dropdown-filter-comp-op"
              @change="filterUpdateCondition(filter, i)"
            >
              <a-select-option v-for="compOp in comparisonOpList" :key="compOp.value" :value="compOp.value" class="">
                {{ compOp.text }}
              </a-select-option>
            </a-select>
            <!--
            todo: filter based on column type

            item-value="value"
            item-text="text"
            :items="filterComparisonOp(filter)" -->

            <!--              <template #item="{ item }"> -->
            <!--                <span class="caption font-weight-regular">{{ item.text }}</span> -->
            <!--              </template> -->
            <!--            </v-select> -->
            <span v-if="['null', 'notnull', 'empty', 'notempty'].includes(filter.comparison_op)" :key="`span${i}`" />
            <a-checkbox
              v-else-if="types[filter.field] === 'boolean'"
              v-model:checked="filter.value"
              dense
              :disabled="filter.readOnly"
              @change="saveOrUpdate(filter, i)"
            />
            <a-input
              v-else
              :key="`${i}_7`"
              v-model:value="filter.value"
              class="nc-filter-value-select"
              :disabled="filter.readOnly"
              @click.stop
              @input="saveOrUpdate(filter, i)"
            />
          </template>
        </template>
      </template>
    </div>

    <div class="flex gap-2 mb-2 mt-4">
      <a-button class="elevation-0 text-capitalize" type="primary" ghost @click.stop="addFilter">
        <div class="flex items-center gap-1">
          <!--      <v-icon small color="grey"> mdi-plus </v-icon> -->
          <MdiPlus />
          <!-- Add Filter -->
          {{ $t('activity.addFilter') }}
        </div>
      </a-button>
      <a-button v-if="!webHook" class="text-capitalize !text-gray-500" @click.stop="addFilterGroup">
        <div class="flex items-center gap-1">
          <!--      <v-icon small color="grey"> mdi-plus </v-icon> -->
          <!--          Add Filter Group -->
          <MdiPlus />
          {{ $t('activity.addFilterGroup') }}
        </div>
      </a-button>
    </div>
    <slot />
  </div>
</template>

<style scoped>
.nc-filter-grid {
  display: grid;
  grid-template-columns: 18px 75px auto auto auto;
  @apply gap-[12px]
  align-items: center;
}

:deep(.ant-select-item-option) {
  @apply "!min-w-full";
}
</style>
