<script setup lang="ts">
import { RelationTypes, UITypes, isVirtualCol } from 'nocodb-sdk'
import { useSharedFormStoreOrThrow } from '#imports'

const {
  sharedFormView,
  submitForm,
  v$,
  formState,
  notFound,
  formColumns,
  submitted,
  secondsRemain,
  passwordDlg,
  password,
  loadSharedView,
  isLoading,
} = useSharedFormStoreOrThrow()

function isRequired(_columnObj: Record<string, any>, required = false) {
  let columnObj = _columnObj
  if (
    columnObj.uidt === UITypes.LinkToAnotherRecord &&
    columnObj.colOptions &&
    columnObj.colOptions.type === RelationTypes.BELONGS_TO
  ) {
    columnObj = formColumns.value?.find((c) => c.id === columnObj.colOptions.fk_child_column_id) as Record<string, any>
  }

  return !!(required || (columnObj && columnObj.rqd && !columnObj.cdf))
}
</script>

<template>
  <div class="nc-form-view md:bg-primary bg-opacity-5 min-h-full flex flex-col nc-form-signin py-15">
    <div
      class="bg-white relative flex flex-col justify-center gap-2 w-full lg:max-w-1/2 max-w-500px m-auto p-8 md:(rounded-lg border-1 border-gray-200 shadow-xl)"
    >
      <template v-if="sharedFormView">
        <general-noco-icon class="color-transition hover:(ring ring-accent)" :class="[isLoading ? 'animated-bg-gradient' : '']" />

        <h1 class="prose-2xl font-bold self-center my-4">{{ sharedFormView?.heading }}</h1>

        <h2 v-if="sharedFormView?.subheading" class="prose-lg text-gray-500 self-center">{{ sharedFormView.subheading }}</h2>

        <a-alert v-if="notFound" type="warning" class="my-4 text-center" message="Not found" />

        <template v-else-if="submitted">
          <div class="flex justify-center">
            <div v-if="sharedFormView" class="min-w-350px mt-3">
              <a-alert
                type="success"
                class="my-4 text-center"
                outlined
                :message="sharedFormView.success_msg || 'Successfully submitted form data'"
              />

              <p v-if="sharedFormView.show_blank_form" class="text-xs text-gray-500 text-center my-4">
                New form will be loaded after {{ secondsRemain }} seconds
              </p>

              <div v-if="sharedFormView.submit_another_form" class="text-center">
                <a-button type="primary" @click="submitted = false"> Submit Another Form</a-button>
              </div>
            </div>
          </div>
        </template>

        <template v-else-if="sharedFormView">
          <div class="nc-form-wrapper">
            <div class="nc-form h-full max-w-3/4 mx-auto">
              <div v-for="(field, index) in formColumns" :key="index" class="flex flex-col my-6 gap-2">
                <div class="flex nc-form-column-label">
                  <SmartsheetHeaderVirtualCell
                    v-if="isVirtualCol(field)"
                    :column="{ ...field, title: field.label || field.title }"
                    :required="isRequired(field, field.required)"
                    :hide-menu="true"
                  />

                  <SmartsheetHeaderCell
                    v-else
                    :column="{ ...field, title: field.label || field.title }"
                    :required="isRequired(field, field.required)"
                    :hide-menu="true"
                  />
                </div>

                <div v-if="isVirtualCol(field)" class="mt-0">
                  <SmartsheetVirtualCell
                    class="mt-0 nc-input"
                    :class="`nc-form-input-${field.title.replaceAll(' ', '')}`"
                    :column="field"
                  />

                  <div v-if="field.description" class="text-gray-500 text-[10px] mb-2 ml-1">{{ field.description }}</div>

                  <template v-if="v$.virtual.$dirty && v$.virtual?.[field.title]">
                    <div v-for="error of v$.virtual[field.title].$errors" :key="error" class="text-xs text-red-500">
                      {{ error.$message }}
                    </div>
                  </template>
                </div>

                <div v-else class="mt-0">
                  <SmartsheetCell
                    v-model="formState[field.title]"
                    class="nc-input"
                    :class="`nc-form-input-${field.title.replaceAll(' ', '')}`"
                    :column="field"
                    :edit-enabled="true"
                  />

                  <div v-if="field.description" class="text-gray-500 text-[10px] mb-2 ml-1">{{ field.description }}</div>

                  <template v-if="v$.localState.$dirty && v$.localState?.[field.title]">
                    <div v-for="error of v$.localState[field.title].$errors" :key="error" class="text-xs text-red-500">
                      {{ error.$message }}
                    </div>
                  </template>
                </div>
              </div>

              <div class="text-center my-9">
                <button type="submit" class="submit" @click="submitForm">
                  {{ $t('general.submit') }}
                </button>
              </div>
            </div>
          </div>
        </template>
      </template>

      <a-modal
        v-model:visible="passwordDlg"
        :closable="false"
        width="28rem"
        centered
        :footer="null"
        :mask-closable="false"
        wrap-class-name="nc-modal-shared-form-password-dlg"
        @close="passwordDlg = false"
      >
        <div class="w-full flex flex-col">
          <a-typography-title :level="4">This shared view is protected</a-typography-title>

          <a-form ref="formRef" :model="{ password }" class="mt-2" @finish="loadSharedView">
            <a-form-item name="password" :rules="[{ required: true, message: $t('msg.error.signUpRules.passwdRequired') }]">
              <a-input-password v-model:value="password" :placeholder="$t('msg.info.signUp.enterPassword')" />
            </a-form-item>

            <!-- Unlock -->
            <a-button type="primary" html-type="submit">{{ $t('general.unlock') }}</a-button>
          </a-form>
        </div>
      </a-modal>
    </div>
  </div>
</template>

<style lang="scss">
.nc-form-view {
  .nc-input {
    @apply w-full rounded p-2 min-h-[40px] flex items-center border-solid border-1 border-primary;
  }

  .submit {
    @apply z-1 relative color-transition rounded p-3 text-white shadow-sm;

    &::after {
      @apply rounded absolute top-0 left-0 right-0 bottom-0 transition-all duration-150 ease-in-out bg-primary;
      content: '';
      z-index: -1;
    }

    &:hover::after {
      @apply transform scale-110 ring ring-accent;
    }

    &:active::after {
      @apply ring ring-accent;
    }
  }
}
</style>
