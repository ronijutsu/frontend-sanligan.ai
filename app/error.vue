<script setup lang="ts">
import type { NuxtError } from '#app'
import { ArrowLeftIcon, HomeIcon, MoonIcon, RotateCcwIcon, SunIcon } from '@lucide/vue'

const props = defineProps<{
  error: NuxtError
}>()

const { isDark, toggle: toggleTheme } = useTheme()
const router = useRouter()

const statusCode = computed(() => props.error?.statusCode ?? 500)
const isDev = import.meta.dev

const isNotFound = computed(() => statusCode.value === 404)
const isUnauthorized = computed(() => statusCode.value === 401)
const isForbidden = computed(() => statusCode.value === 403)
const isRateLimited = computed(() => statusCode.value === 429)
const isClientError = computed(() => statusCode.value >= 400 && statusCode.value < 500)
const isServerError = computed(() => statusCode.value >= 500)

const copy = computed(() => {
  switch (statusCode.value) {
    case 400:
      return {
        eyebrow: 'Bad request',
        title: 'That request didn’t look right',
        body: 'Check the link or what was submitted, then try again. If you followed a link inside Batayan, going back usually fixes it.',
      }
    case 401:
      return {
        eyebrow: 'Sign-in required',
        title: 'Please sign in to continue',
        body: 'Your session may have expired. Sign in again to pick up where you left off.',
      }
    case 403:
      return {
        eyebrow: 'Access denied',
        title: 'You don’t have access to this',
        body: 'This page or action belongs to a different role or workspace. Ask an administrator if you think this is a mistake.',
      }
    case 404:
      return {
        eyebrow: 'Page not found',
        title: 'This page went missing',
        body: 'The link may be broken, the page may have moved, or it never existed. The rest of your workspace is untouched.',
      }
    case 429:
      return {
        eyebrow: 'Too many requests',
        title: 'Slow down a little',
        body: 'Too many attempts in a short time. Wait a minute, then try again.',
      }
    default:
      if (isClientError.value) {
        return {
          eyebrow: `Error ${statusCode.value}`,
          title: 'Something looks off with that request',
          body: props.error?.statusMessage || 'The request could not be completed. Try again, or go back to where you were.',
        }
      }
      return {
        eyebrow: `Error ${statusCode.value}`,
        title: 'Something went wrong on our end',
        body: 'This wasn’t caused by anything you did. Try again — if it keeps happening, contact support and mention the error code below.',
      }
  }
})

useHead({
  title: () => `${statusCode.value} — ${copy.value.title} | Batayan`,
  meta: [{ name: 'robots', content: 'noindex, nofollow' }],
})

/** Home goes through `/` so the auth + onboarding middleware pick the right landing. */
async function goHome() {
  await clearError({ redirect: '/' })
}

async function goLogin() {
  await clearError({ redirect: '/login' })
}

async function retry() {
  await clearError()
  await reloadNuxtApp({ force: true })
}

async function goBack() {
  await clearError()
  if (window.history.length > 1) {
    router.back()
  } else {
    await navigateTo('/')
  }
}
</script>

<template>
  <div class="flex min-h-dvh flex-col bg-background text-foreground">
    <header class="flex h-14 shrink-0 items-center gap-3 px-4">
      <span class="flex items-center gap-2 font-heading font-semibold tracking-tight">
        <span class="size-2.5 rounded-full bg-primary" aria-hidden="true" />
        Batayan
      </span>

      <div class="ml-auto flex items-center gap-1">
        <button
          type="button"
          aria-label="Toggle theme"
          class="flex size-8 items-center justify-center rounded-full text-muted-foreground transition-colors hover:bg-muted hover:text-foreground"
          @click="toggleTheme"
        >
          <component :is="isDark ? SunIcon : MoonIcon" class="size-4" />
        </button>
      </div>
    </header>

    <main class="flex flex-1 items-center justify-center px-4 pb-16 pt-4">
      <div class="w-full max-w-lg text-center" role="alert">
        <p class="font-heading text-7xl font-semibold tracking-tight text-primary sm:text-8xl" aria-hidden="true">
          {{ statusCode }}
        </p>

        <p class="mt-4 flex items-center justify-center gap-2 text-xs font-semibold uppercase tracking-[0.18em] text-muted-foreground">
          <span class="h-px w-8 bg-border" aria-hidden="true" />
          {{ copy.eyebrow }}
          <span class="h-px w-8 bg-border" aria-hidden="true" />
        </p>

        <h1 class="mx-auto mt-3 font-heading text-2xl font-medium leading-[1.15] tracking-tight sm:text-3xl">
          {{ copy.title }}
        </h1>

        <p class="mx-auto mt-3 max-w-sm text-sm leading-relaxed text-muted-foreground">
          {{ copy.body }}
        </p>

        <div class="mt-8 flex flex-col items-center justify-center gap-2 sm:flex-row">
          <template v-if="isUnauthorized">
            <Button class="w-full sm:w-auto" @click="goLogin">
              <HomeIcon data-icon="inline-start" />
              Go to sign in
            </Button>
            <Button class="w-full sm:w-auto" variant="outline" @click="goBack">
              <ArrowLeftIcon data-icon="inline-start" />
              Go back
            </Button>
          </template>
          <template v-else-if="isServerError">
            <Button class="w-full sm:w-auto" @click="retry">
              <RotateCcwIcon data-icon="inline-start" />
              Try again
            </Button>
            <Button class="w-full sm:w-auto" variant="outline" @click="goHome">
              <HomeIcon data-icon="inline-start" />
              Back to dashboard
            </Button>
          </template>
          <template v-else-if="isNotFound || isForbidden || isRateLimited">
            <Button class="w-full sm:w-auto" @click="goHome">
              <HomeIcon data-icon="inline-start" />
              Back to dashboard
            </Button>
            <Button class="w-full sm:w-auto" variant="outline" @click="goBack">
              <ArrowLeftIcon data-icon="inline-start" />
              Go back
            </Button>
          </template>
          <template v-else>
            <Button class="w-full sm:w-auto" @click="retry">
              <RotateCcwIcon data-icon="inline-start" />
              Try again
            </Button>
            <Button class="w-full sm:w-auto" variant="outline" @click="goBack">
              <ArrowLeftIcon data-icon="inline-start" />
              Go back
            </Button>
          </template>
        </div>

        <details v-if="isDev && error?.message" class="surface-inset mx-auto mt-8 max-w-md px-4 py-3 text-left">
          <summary class="cursor-pointer text-xs font-medium text-muted-foreground">
            Developer details
          </summary>
          <p class="mt-2 break-words font-mono text-xs leading-relaxed">
            {{ error.message }}
          </p>
          <p v-if="error.stack" class="mt-1 break-words font-mono text-xs leading-relaxed text-muted-foreground">
            {{ error.stack }}
          </p>
        </details>

        <p v-else-if="isServerError" class="mt-6 text-xs leading-relaxed text-muted-foreground">
          Error code {{ statusCode }} — mention it if you contact support.
        </p>
      </div>
    </main>
  </div>
</template>
