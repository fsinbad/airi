<script setup lang="ts">
import type { Voice } from '@proj-airi/stage-ui/constants'
import type { RemovableRef } from '@vueuse/core'
import type { UnElevenLabsOptions } from '@xsai-ext/providers-local'
import type { SpeechProviderWithExtraOptions } from '@xsai-ext/shared-providers'

import { Collapsable, Range } from '@proj-airi/stage-ui/components'
import { voiceMap } from '@proj-airi/stage-ui/constants'
import { useProvidersStore, useSpeechStore } from '@proj-airi/stage-ui/stores'
import { useToggle } from '@vueuse/core'
import { generateSpeech } from '@xsai/generate-speech'
import { storeToRefs } from 'pinia'
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()
const providersStore = useProvidersStore()
const speechStore = useSpeechStore()
const { providers } = storeToRefs(providersStore) as { providers: RemovableRef<Record<string, any>> }

// For playground
const testText = ref('Hello! This is a test of the ElevenLabs voice synthesis.')
const isGenerating = ref(false)
const audioUrl = ref('')
const errorMessage = ref('')
const audioPlayer = ref<HTMLAudioElement | null>(null)

// Get provider metadata
const providerId = 'elevenlabs'
const providerMetadata = computed(() => providersStore.getProviderMetadata(providerId))

const apiKey = computed({
  get: () => providers.value[providerId]?.apiKey || '',
  set: (value) => {
    if (!providers.value[providerId])
      providers.value[providerId] = {}

    providers.value[providerId].apiKey = value
  },
})

const baseUrl = computed({
  get: () => providers.value[providerId]?.baseUrl || providerMetadata.value?.defaultOptions?.baseUrl || '',
  set: (value) => {
    if (!providers.value[providerId])
      providers.value[providerId] = {}

    providers.value[providerId].baseUrl = value
  },
})

// Voice settings as individual computed properties
const similarityBoost = computed({
  get: () => (providers.value[providerId]?.voiceSettings as any)?.similarityBoost ?? 0.75,
  set: (value) => {
    if (!providers.value[providerId])
      providers.value[providerId] = {}
    if (!providers.value[providerId].voiceSettings)
      providers.value[providerId].voiceSettings = {}

    providers.value[providerId].voiceSettings.similarityBoost = value
  },
})

const stability = computed({
  get: () => (providers.value[providerId]?.voiceSettings as any)?.stability ?? 0.5,
  set: (value) => {
    if (!providers.value[providerId])
      providers.value[providerId] = {}
    if (!providers.value[providerId].voiceSettings)
      providers.value[providerId].voiceSettings = {}

    providers.value[providerId].voiceSettings.stability = value
  },
})

const speed = computed({
  get: () => (providers.value[providerId]?.voiceSettings as any)?.speed ?? 1.0,
  set: (value) => {
    if (!providers.value[providerId])
      providers.value[providerId] = {}
    if (!providers.value[providerId].voiceSettings)
      providers.value[providerId].voiceSettings = {}

    providers.value[providerId].voiceSettings.speed = value
  },
})

const style = computed({
  get: () => (providers.value[providerId]?.voiceSettings as any)?.style ?? 0,
  set: (value) => {
    if (!providers.value[providerId])
      providers.value[providerId] = {}
    if (!providers.value[providerId].voiceSettings)
      providers.value[providerId].voiceSettings = {}

    providers.value[providerId].style = value
  },
})

const useSpeakerBoost = computed({
  get: () => (providers.value[providerId]?.voiceSettings as any)?.useSpeakerBoost !== false,
  set: (value) => {
    if (!providers.value[providerId])
      providers.value[providerId] = {}
    if (!providers.value[providerId].voiceSettings)
      providers.value[providerId].voiceSettings = {}

    providers.value[providerId].voiceSettings.useSpeakerBoost = value
  },
})

// Speech settings
const selectedLanguage = ref(speechStore.selectedLanguage)
const selectedVoice = ref(speechStore.voiceName)
const availableVoices = computed(() => speechStore.availableVoicesForLanguage)

const advancedVisible = ref(false)
const toggleAdvancedVisible = useToggle(advancedVisible)

onMounted(() => {
  providersStore.initializeProvider(providerId)

  // Initialize refs with current values
  apiKey.value = providers.value[providerId]?.apiKey || ''
  baseUrl.value = providers.value[providerId]?.baseUrl || providerMetadata.value?.defaultOptions?.baseUrl || ''

  // Initialize voice settings refs
  if (providers.value[providerId]?.voiceSettings) {
    similarityBoost.value = (providers.value[providerId].voiceSettings as any)?.similarityBoost ?? 0.75
    stability.value = (providers.value[providerId].voiceSettings as any)?.stability ?? 0.5
    speed.value = (providers.value[providerId].voiceSettings as any)?.speed ?? 1.0
    style.value = (providers.value[providerId].voiceSettings as any)?.style ?? 0
    useSpeakerBoost.value = (providers.value[providerId].voiceSettings as any)?.useSpeakerBoost !== false
  }

  // Load voices if provider is configured
  if (providersStore.configuredProviders[providerId]) {
    speechStore.loadVoicesForProvider(providerId)
  }
})

// Watch all settings and update the provider configuration
watch([apiKey, baseUrl, similarityBoost, stability, speed, style, useSpeakerBoost], () => {
  providers.value[providerId] = {
    ...providers.value[providerId],
    apiKey: apiKey.value,
    baseUrl: baseUrl.value || providerMetadata.value?.defaultOptions?.baseUrl || '',
    voiceSettings: {
      similarityBoost: similarityBoost.value,
      stability: stability.value,
      speed: speed.value,
      style: style.value,
      useSpeakerBoost: useSpeakerBoost.value,
    },
  }
})

// Function to generate speech
async function generateTestSpeech() {
  if (!testText.value.trim())
    return

  const provider = providersStore.getProviderInstance(providerId) as SpeechProviderWithExtraOptions<string, UnElevenLabsOptions>
  if (!provider) {
    console.error('Failed to initialize speech provider')
    return
  }

  isGenerating.value = true
  errorMessage.value = ''

  try {
    // Stop any currently playing audio
    if (audioUrl.value) {
      stopTestAudio()
    }

    const response = await generateSpeech({
      ...provider.speech('eleven_multilingual_v2', {
        voiceSettings: {
          stability: stability.value,
          similarityBoost: similarityBoost.value,
          // @ts-expect-error -- missing type
          speed: speed.value,
          style: style.value,
          useSpeakerBoost: useSpeakerBoost.value,
        },
      }),
      input: testText.value,
      voice: voiceMap[selectedVoice.value as Voice],
    })

    // Convert the response to a blob and create an object URL
    audioUrl.value = URL.createObjectURL(new Blob([response]))

    // Play the audio
    setTimeout(() => {
      if (audioPlayer.value) {
        audioPlayer.value.play()
      }
    }, 100)
  }
  catch (error) {
    console.error('Error generating speech:', error)
    errorMessage.value = error instanceof Error ? error.message : 'An unknown error occurred'
  }
  finally {
    isGenerating.value = false
  }
}

// Function to stop audio playback
function stopTestAudio() {
  if (audioPlayer.value) {
    audioPlayer.value.pause()
    audioPlayer.value.currentTime = 0
  }

  // Clean up the object URL to prevent memory leaks
  if (audioUrl.value) {
    URL.revokeObjectURL(audioUrl.value)
    audioUrl.value = ''
  }
}

// Clean up when component is unmounted
onUnmounted(() => {
  if (audioUrl.value) {
    URL.revokeObjectURL(audioUrl.value)
  }
})

function handleResetVoiceSettings() {
  providers.value[providerId] = {
    ...(providerMetadata.value?.defaultOptions as any),
  }
}
</script>

<template>
  <div flex="~ row" items-center gap-2>
    <button @click="router.back()">
      <div i-solar:alt-arrow-left-line-duotone text-2xl />
    </button>
    <h1 relative>
      <div absolute left-0 top-0 translate-y="[-80%]">
        <span text="neutral-300 dark:neutral-500">Provider</span>
      </div>
      <div text-3xl font-semibold>
        {{ providerMetadata?.localizedName }}
      </div>
    </h1>
  </div>
  <div flex="~ col md:row gap-6">
    <div bg="neutral-50 dark:[rgba(0,0,0,0.3)]" rounded-xl p-4 flex="~ col gap-6" w="full md:40%">
      <div>
        <div flex="~ col gap-6">
          <div>
            <h2 class="text-lg text-neutral-500 md:text-2xl dark:text-neutral-400">
              Basic
            </h2>
            <div text="neutral-400 dark:neutral-500">
              <span>Essential settings</span>
            </div>
          </div>
          <div max-w-full>
            <label flex="~ col gap-4">
              <div>
                <div class="flex items-center gap-1 text-sm font-medium">
                  API Key
                  <span class="text-red-500">*</span>
                </div>
                <div class="text-xs text-neutral-500 dark:text-neutral-400" text-nowrap>
                  API Key for {{ providerMetadata?.localizedName }}
                </div>
              </div>
              <input
                v-model="apiKey"
                type="password"
                border="neutral-300 dark:neutral-800 solid 2 focus:neutral-400 dark:focus:neutral-600"
                transition="border duration-250 ease-in-out"
                w-full rounded-lg px-2 py-1 text-nowrap text-sm outline-none
                bg="neutral-200 dark:neutral-800"
                placeholder="sk-"
              >
            </label>
          </div>
        </div>
      </div>

      <div flex="~ col gap-6">
        <h2 class="text-lg text-neutral-500 md:text-2xl dark:text-neutral-400">
          Voice Settings
        </h2>
        <div flex="~ col gap-4">
          <label flex="~ col gap-2">
            <div flex="~ row" items-center gap-2>
              <div flex="1">
                <div class="flex items-center gap-1 text-sm font-medium">
                  Similarity Boost
                </div>
                <div class="text-xs text-neutral-500 dark:text-neutral-400">
                  Voice similarity adherence
                </div>
              </div>
              <span font-mono>{{ similarityBoost.toFixed(2) }}</span>
            </div>
            <div flex="~ row" items-center gap-2>
              <Range v-model="similarityBoost" :min="0" :max="1" :step="0.01" w-full />
            </div>
          </label>

          <label flex="~ col gap-4">
            <div flex="~ row" items-center gap-2>
              <div flex="1">
                <div class="flex items-center gap-1 text-sm font-medium">
                  Stability
                </div>
                <div class="text-xs text-neutral-500 dark:text-neutral-400">
                  Voice stability and randomness
                </div>
              </div>
              <span font-mono>{{ stability.toFixed(2) }}</span>
            </div>
            <div flex="~ row" items-center gap-2>
              <Range v-model="stability" :min="0" :max="1" :step="0.01" w-full />
            </div>
          </label>

          <label flex="~ col gap-4">
            <div flex="~ row" items-center gap-2>
              <div flex="1">
                <div class="flex items-center gap-1 text-sm font-medium">
                  Speed
                </div>
                <div class="text-xs text-neutral-500 dark:text-neutral-400">
                  Speech generation speed
                </div>
              </div>
              <span font-mono>{{ speed.toFixed(2) }}</span>
            </div>
            <div flex="~ row" items-center gap-2>
              <Range v-model="speed" :min="0.7" :max="1.2" :step="0.01" w-full />
            </div>
          </label>

          <label flex="~ col gap-4">
            <div flex="~ row" items-center gap-2>
              <div flex="1">
                <div class="flex items-center gap-1 text-sm font-medium">
                  Style
                </div>
                <div class="text-xs text-neutral-500 dark:text-neutral-400">
                  Voice style exaggeration
                </div>
              </div>
              <span font-mono>{{ style.toFixed(2) }}</span>
            </div>
            <div flex="~ row" items-center gap-2>
              <Range v-model="style" :min="0" :max="1" :step="0.01" w-full />
            </div>
          </label>

          <label flex="~ col gap-4">
            <div flex="~ row" items-center gap-2>
              <div flex="1">
                <div class="flex items-center gap-1 text-sm font-medium">
                  Speaker Boost
                </div>
                <div class="text-xs text-neutral-500 dark:text-neutral-400">
                  Enhance speaker similarity
                </div>
              </div>
              <span font-mono>{{ useSpeakerBoost ? 'On' : 'Off' }}</span>
            </div>
            <div flex="~ row" items-center gap-2>
              <input v-model="useSpeakerBoost" type="checkbox">
            </div>
          </label>
        </div>
      </div>

      <div>
        <Collapsable w-full>
          <template #trigger="slotProps">
            <button
              transition="all ease-in-out duration-250"
              w-full flex items-center gap-1.5 outline-none
              class="[&_.provider-icon]:grayscale-100 [&_.provider-icon]:hover:grayscale-0"
              @click="() => slotProps.setVisible(!slotProps.visible) && toggleAdvancedVisible()"
            >
              <h2 class="text-lg text-neutral-500 md:text-2xl dark:text-neutral-400">
                <span>Advanced</span>
              </h2>
              <div transform transition="transform duration-250" :class="{ 'rotate-180': slotProps.visible }">
                <div i-solar:alt-arrow-down-bold-duotone />
              </div>
            </button>
          </template>
          <div mt-4>
            <label flex="~ col gap-4">
              <div>
                <div class="flex items-center gap-1 text-sm font-medium">
                  Base URL
                  <span class="text-red-500">*</span>
                </div>Base URL
                <div class="text-xs text-neutral-500 dark:text-neutral-400" text-nowrap>
                  Custom base URL (optional)
                </div>
              </div>
              <input
                v-model="baseUrl"
                border="neutral-300 dark:neutral-800 solid 2 focus:neutral-400 dark:focus:neutral-600"
                transition="border duration-250 ease-in-out"
                w-full rounded-lg px-2 py-1 text-nowrap text-sm outline-none
                bg="neutral-200 dark:neutral-800"
                :placeholder="providerMetadata?.defaultOptions?.baseUrl as string || ''"
              >
            </label>

            <div mt-4>
              <button
                border="neutral-300 dark:neutral-800 solid 1"
                transition="border duration-250 ease-in-out" rounded
                px-4 py-2 text-sm
                @click="handleResetVoiceSettings"
              >
                Reset Voice Settings
              </button>
            </div>
          </div>
        </Collapsable>
      </div>
    </div>
    <div flex="~ col gap-6" w="full md:60%">
      <div rounded-xl>
        <h2 class="mb-4 text-lg text-neutral-500 md:text-2xl dark:text-neutral-400">
          Voice Playground
        </h2>
        <div flex="~ col gap-4">
          <textarea
            v-model="testText" placeholder="Enter text to test the voice..."
            border="neutral-300 dark:neutral-800 solid 2 focus:neutral-400 dark:focus:neutral-600"
            transition="border duration-250 ease-in-out"
            h-24 w-full rounded-lg px-3 py-2 text-sm outline-none
            bg="neutral-200 dark:neutral-800"
          />
          <div flex="~ col gap-6">
            <label grid="~ cols-2 gap-4">
              <div>
                <div class="flex items-center gap-1 text-sm font-medium">
                  Language
                </div>
                <div class="text-xs text-neutral-500 dark:text-neutral-400">
                  Select voice language
                </div>
              </div>
              <select
                v-model="selectedLanguage"
                border="neutral-300 dark:neutral-800 solid 2 focus:neutral-400 dark:focus:neutral-600"
                transition="border duration-250 ease-in-out" w-full rounded-lg px-2 py-1 text-nowrap text-sm outline-none
              >
                <option v-for="language in speechStore.availableLanguages" :key="language" :value="language">
                  {{ language }}
                </option>
              </select>
            </label>

            <label grid="~ cols-2 gap-4">
              <div>
                <div class="flex items-center gap-1 text-sm font-medium">
                  Voice
                </div>
                <div class="text-xs text-neutral-500 dark:text-neutral-400">
                  Select preferred voice
                </div>
              </div>
              <select
                v-model="selectedVoice"
                border="neutral-300 dark:neutral-800 solid 2 focus:neutral-400 dark:focus:neutral-600"
                transition="border duration-250 ease-in-out" w-full rounded-lg px-2 py-1 text-nowrap text-sm outline-none
              >
                <option v-for="voice in availableVoices" :key="voice.id" :value="voice.name">
                  {{ voice.name }}
                </option>
              </select>
            </label>
          </div>
          <div flex="~ row" gap-4>
            <button
              border="neutral-800 dark:neutral-200 solid 2"
              transition="border duration-250 ease-in-out" rounded-lg px-4
              text="neutral-100 dark:neutral-900"
              py-2 text-sm :disabled="isGenerating || !testText.trim() || !apiKey"
              :class="{ 'opacity-50 cursor-not-allowed': isGenerating || !testText.trim() || !apiKey }"
              bg="neutral-700 dark:neutral-300" @click="generateTestSpeech"
            >
              <div flex="~ row" items-center gap-2>
                <div i-solar:play-circle-bold-duotone />
                <span>{{ isGenerating ? 'Generating...' : 'Test Voice' }}</span>
              </div>
            </button>
            <button
              v-if="audioUrl" border="primary-300 dark:primary-800 solid 2"
              transition="border duration-250 ease-in-out" rounded-lg px-4 py-2 text-sm @click="stopTestAudio"
            >
              <div flex="~ row" items-center gap-2>
                <div i-solar:stop-circle-bold-duotone />
                <span>Stop</span>
              </div>
            </button>
          </div>
          <div v-if="!apiKey" class="mt-2 text-sm text-red-500">
            Please enter an API key to test the voice.
          </div>
          <div v-if="errorMessage" class="mt-2 text-sm text-red-500">
            {{ errorMessage }}
          </div>
          <audio v-if="audioUrl" ref="audioPlayer" :src="audioUrl" controls class="mt-2 w-full" />
        </div>
      </div>
    </div>
  </div>
  <div text="neutral-100/50 dark:neutral-500/20" pointer-events-none fixed bottom-0 right-0 translate-x-10 translate-y-10>
    <div text="40" :class="providerMetadata?.icon" />
  </div>
</template>
