<script setup lang="ts">
import { validatePassword } from 'nocodb-sdk'
import {
  definePageMeta,
  extractSdkResponseErrorMsg,
  isEmail,
  navigateTo,
  reactive,
  ref,
  useApi,
  useGlobal,
  useI18n,
  useNuxtApp,
  useRoute,
} from '#imports'

definePageMeta({
  requiresAuth: false,
  title: 'general.signUp',
})

const { $e } = useNuxtApp()

const route = useRoute()

const { appInfo, signIn } = useGlobal()

const { api, isLoading } = useApi()

const { t } = useI18n()

const formValidator = ref()

let error = $ref<string | null>(null)

const subscribe = ref(false)

const form = reactive({
  email: '',
  password: '',
})

const formRules = {
  email: [
    // E-mail is required
    { required: true, message: t('msg.error.signUpRules.emailReqd') },
    // E-mail must be valid format
    {
      validator: (_: unknown, v: string) => {
        return new Promise((resolve, reject) => {
          if (!v?.length || isEmail(v)) return resolve(true)
          reject(new Error(t('msg.error.signUpRules.emailInvalid')))
        })
      },
      message: t('msg.error.signUpRules.emailInvalid'),
    },
  ],
  password: [
    {
      validator: (_: unknown, v: string) => {
        return new Promise((resolve, reject) => {
          const { error, valid } = validatePassword(v)
          if (valid) return resolve(true)
          reject(new Error(error))
        })
      },
    },
  ],
}

async function signUp() {
  if (!formValidator.value.validate()) return

  resetError()

  const data: any = {
    ...form,
    token: route.params.token,
  }

  data.ignore_subscribe = !subscribe.value

  api.auth
    .signup(data)
    .then(async ({ token }) => {
      signIn(token!)

      await navigateTo('/')

      $e('a:auth:sign-up')
    })
    .catch(async (err) => {
      error = await extractSdkResponseErrorMsg(err)
    })
}

function resetError() {
  if (error) error = null
}
</script>

<template>
  <NuxtLayout>
    <div class="md:bg-primary bg-opacity-5 signup h-full min-h-[600px] flex flex-col justify-center items-center">
      <div
        class="bg-white mt-[60px] relative flex flex-col justify-center gap-2 w-full max-w-[500px] mx-auto p-8 md:(rounded-lg border-1 border-gray-200 shadow-xl)"
      >
        <general-noco-icon class="color-transition hover:(ring ring-accent)" :class="[isLoading ? 'animated-bg-gradient' : '']" />

        <h1 class="prose-2xl font-bold self-center my-4">
          {{ $t('general.signUp') }}
          {{ $route.query.redirect_to === '/referral' ? '& REFER' : '' }}
          {{ $route.query.redirect_to === '/pricing' ? '& BUY' : '' }}
        </h1>

        <h2 v-if="appInfo.firstUser" class="prose !text-primary font-semibold self-center">
          {{ $t('msg.info.signUp.superAdmin') }}
        </h2>

        <a-form ref="formValidator" :model="form" layout="vertical" no-style @finish="signUp">
          <Transition name="layout">
            <div v-if="error" class="self-center mb-4 bg-red-500 text-white rounded-lg w-3/4 mx-auto p-1">
              <div class="flex items-center gap-2 justify-center">
                <MaterialSymbolsWarning />
                <div class="break-words">{{ error }}</div>
              </div>
            </div>
          </Transition>

          <a-form-item :label="$t('labels.email')" name="email" :rules="formRules.email">
            <a-input v-model:value="form.email" size="large" :placeholder="$t('msg.info.signUp.workEmail')" @focus="resetError" />
          </a-form-item>

          <a-form-item :label="$t('labels.password')" name="password" :rules="formRules.password">
            <a-input-password
              v-model:value="form.password"
              size="large"
              class="password"
              :placeholder="$t('msg.info.signUp.enterPassword')"
              @focus="resetError"
            />
          </a-form-item>

          <div class="self-center flex flex-col flex-wrap gap-4 items-center mt-4">
            <button class="submit" type="submit">
              <span class="flex items-center gap-2">
                <MaterialSymbolsRocketLaunchOutline />

                {{ $t('general.signUp') }}
              </span>
            </button>

            <a
              v-if="appInfo.googleAuthEnabled"
              :href="`${api.instance.defaults.baseURL}/auth/google`"
              class="submit after:(!bg-white) !text-primary border-1 border-primary !no-underline"
            >
              <span class="flex items-center gap-2">
                <LogosGoogleGmail />

                Sign up with Google
              </span>
            </a>

            <div class="flex items-center gap-2">
              <a-switch
                v-model:checked="subscribe"
                size="small"
                class="my-1 hover:(ring ring-accent) focus:(!ring !ring-accent)"
              />
              <div class="prose-xs text-gray-500">Subscribe to our weekly newsletter</div>
            </div>

            <div class="text-end prose-sm">
              {{ $t('msg.info.signUp.alreadyHaveAccount') }}

              <nuxt-link to="/signin">{{ $t('general.signIn') }}</nuxt-link>
            </div>
          </div>
        </a-form>
      </div>

      <div class="prose-sm mt-4 text-gray-500">
        By signing up, you agree to the
        <a class="prose-sm !text-gray-500 underline" target="_blank" href="https://nocodb.com/policy-nocodb">Terms of Service</a>
      </div>
    </div>
  </NuxtLayout>
</template>

<style lang="scss">
.signup {
  .ant-input-affix-wrapper,
  .ant-input {
    @apply !appearance-none my-1 border-1 border-solid border-primary/50 rounded;
  }

  .password {
    input {
      @apply !border-none !m-0;
    }
  }

  .submit {
    @apply z-1 relative color-transition rounded p-3 text-white shadow;

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
