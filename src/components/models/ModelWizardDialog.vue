<script setup lang="ts">
import { ref, watch, computed } from 'vue'
import { nanoid } from 'nanoid'
import { useModelConfigStore } from '@/stores/models'
import { useProxyStore } from '@/stores/proxy'
import { Button } from '@/components/ui/button'
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '@/components/ui/dialog'
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import { Checkbox } from '@/components/ui/checkbox'
import { openUrl } from '@tauri-apps/plugin-opener'
import { Wand2, Loader2, ExternalLink, ShieldCheck, Cloud, Key, Box } from 'lucide-vue-next'
import { toast } from 'vue-sonner'
import { useI18n } from 'vue-i18n'
import { providerList as providers } from '@/configs/providers'

const { t } = useI18n()
const modelStore = useModelConfigStore()
const proxyStore = useProxyStore()

const emit = defineEmits<{
  added: [ids: string[]]
}>()

const isOpen = ref(false)
const isLoading = ref(false)
const models = ref<string[]>([])
const provider = ref('')
const apiKey = ref('')
const selectedModels = ref<Set<string>>(new Set())
const modelFilter = ref('')
const currentId = ref('')

// Models filtered by the search box
const filteredModels = computed(() => {
  const query = modelFilter.value.trim().toLowerCase()
  if (!query) return models.value
  return models.value.filter(m => m.toLowerCase().includes(query))
})

const selectedCount = computed(() => selectedModels.value.size)

// Whether every currently visible (filtered) model is selected
const allFilteredSelected = computed(
  () => filteredModels.value.length > 0 && filteredModels.value.every(m => selectedModels.value.has(m))
)

const toggleModel = (name: string, checked: boolean) => {
  if (checked) {
    selectedModels.value.add(name)
  } else {
    selectedModels.value.delete(name)
  }
}

const onModelToggle = (name: string, value: boolean | 'indeterminate') => {
  if (value !== 'indeterminate') toggleModel(name, value)
}

const toggleSelectAll = (value: boolean | 'indeterminate') => {
  if (value === 'indeterminate') return
  for (const m of filteredModels.value) {
    toggleModel(m, value)
  }
}

watch(isOpen, (newVal) => {
  if (newVal) {
    currentId.value = nanoid(6)
    provider.value = ''
    apiKey.value = ''
    selectedModels.value = new Set()
    modelFilter.value = ''
    models.value = []
  } else if (modelStore.getById(currentId.value)) {
    // Remove leftover temp config when the wizard is closed/cancelled
    modelStore.removeConfig(currentId.value)
  }
})

// A fetched model list belongs to one provider only - discard it on provider change
watch(provider, () => {
  models.value = []
  selectedModels.value = new Set()
})

const handleVerify = async () => {
  if (!provider.value || !apiKey.value) return

  isLoading.value = true
  const p = providers.find(item => item.name === provider.value)!

  // Add or update temp config
  const existing = modelStore.getById(currentId.value)
  if (existing) {
    modelStore.updateConfig(currentId.value, {
      baseUrl: p.url,
      apiKey: apiKey.value
    })
  } else {
    modelStore.addConfig({
      id: currentId.value,
      baseUrl: p.url,
      apiKey: apiKey.value,
      model: ''
    })
  }

  // Force save to ensure backend can read it
  await modelStore.saveNow()

  // Wait a moment for the file to be written
  await new Promise(resolve => setTimeout(resolve, 300))

  // Ensure proxy is running
  if (proxyStore.status !== 'running') {
    await proxyStore.start()
  }

  try {
    const response = await fetch(`http://localhost:${proxyStore.port}/${currentId.value}/v1/models`)

    if (!response.ok) {
      const text = await response.text()
      throw new Error(`Status ${response.status}: ${text}`)
    }

    const data = await response.json()
    // Handle OpenAI format { data: [{ id: '...' }, ...] }
    if (data.data && Array.isArray(data.data)) {
      const ids = (data.data as { id: string }[]).map(m => m.id)
      models.value = [...new Set(ids)]
      selectedModels.value = new Set()
      modelFilter.value = ''
      toast.success(t('models.wizard.verifySuccess'))
    } else {
      throw new Error('Invalid response format')
    }
  } catch (e) {
    console.error(e)
    models.value = []
    selectedModels.value = new Set()
    toast.error(t('models.wizard.verifyFail') + ': ' + String(e))
  } finally {
    isLoading.value = false
  }
}

const handleFinish = () => {
  if (!provider.value) return
  const p = providers.find(item => item.name === provider.value)!

  const chosen = [...selectedModels.value]
  // Skip duplicates: a config with the same baseUrl + model already exists
  const toAdd = chosen.filter(m => !modelStore.configs.some(c => c.baseUrl === p.url && c.model === m))
  const skipped = chosen.length - toAdd.length

  const createdIds = toAdd.map(m => {
    const id = nanoid(6)
    modelStore.addConfig({
      id,
      baseUrl: p.url,
      apiKey: apiKey.value,
      model: m
    })
    return id
  })

  // Always remove the temporary config used for verification
  if (modelStore.getById(currentId.value)) {
    modelStore.removeConfig(currentId.value)
  }

  if (toAdd.length > 0) {
    // Save again so the backend can read all new configs immediately
    modelStore.saveNow()
    emit('added', createdIds)
    toast.success(t('models.wizard.added', { count: toAdd.length }))
  }
  if (skipped > 0) {
    toast.info(t('models.wizard.skipped', { count: skipped }))
  }
  isOpen.value = false
}

const openProviderUrl = () => {
  const p = providers.find(item => item.name === provider.value)
  if (p) {
    openUrl(p.keyUrl)
  }
}

</script>

<template>
  <Dialog v-model:open="isOpen">
    <DialogTrigger as-child>
      <Button variant="secondary">
        <Wand2 class="w-4 h-4" />
        {{
          t('models.wizard.title')
        }}
      </Button>
    </DialogTrigger>
    <DialogContent>
      <DialogHeader>
        <DialogTitle>{{ t('models.wizard.title') }}</DialogTitle>
        <DialogDescription>
          {{ t('models.wizard.description') }}
        </DialogDescription>
      </DialogHeader>

      <div class="grid gap-3">
        <!-- Provider -->
        <div class="grid gap-2">
          <Label>
            <Cloud class="size-4" />
            {{ t('models.wizard.provider') }}
          </Label>
          <div class="flex items-center">
            <Select v-model="provider">
              <SelectTrigger>
                <SelectValue :placeholder="t('models.wizard.selectProvider')" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem v-for="p in providers" :key="p.name" :value="p.name">
                  {{ p.name }}
                </SelectItem>
              </SelectContent>
            </Select>
            <Button variant="link" class="text-xs" @click="openProviderUrl">
              {{ t('models.wizard.getApiKey') }}
              <ExternalLink class="w-3 h-3" />
            </Button>
          </div>
        </div>

        <!-- API Key -->
        <div class="grid gap-2" v-if="provider">
          <Label>
            <Key class="size-4" />
            {{ t('models.form.apiKey') }}
          </Label>

          <Input v-model="apiKey" type="password" :placeholder="t('models.form.apiKeyPlaceholder')" />
        </div>

        <!-- Verify Button -->
        <Button v-if="provider && apiKey" @click="handleVerify" :disabled="isLoading" variant="outline" class="w-full">
          <Loader2 v-if="isLoading" class="animate-spin" />
          <ShieldCheck v-else />
          {{ t('models.wizard.verify') }}
        </Button>

        <!-- Model Selection -->
        <div class="grid gap-2" v-if="models.length > 0">
          <Label>
            <Box class="size-4" />
            {{ t('models.wizard.selectModels') }}
          </Label>
          <Input v-model="modelFilter" :placeholder="t('models.wizard.searchModels')" />
          <div class="flex items-center justify-between text-sm text-muted-foreground">
            <label class="flex items-center gap-2 cursor-pointer">
              <Checkbox :model-value="allFilteredSelected" @update:model-value="toggleSelectAll" />
              {{ t('models.wizard.selectAll') }}
            </label>
            <span>{{ t('models.wizard.selectedCount', selectedCount) }}</span>
          </div>
          <div class="max-h-60 overflow-y-auto rounded-md border p-2 grid gap-1">
            <label v-for="m in filteredModels" :key="m"
              class="flex items-center gap-2 px-2 py-1.5 rounded hover:bg-muted cursor-pointer text-sm">
              <Checkbox :model-value="selectedModels.has(m)" @update:model-value="onModelToggle(m, $event)" />
              <span class="truncate">{{ m }}</span>
            </label>
            <div v-if="filteredModels.length === 0" class="text-sm text-muted-foreground p-2">
              {{ t('models.wizard.noModelResults') }}
            </div>
          </div>
        </div>
      </div>

      <DialogFooter>
        <Button @click="handleFinish" :disabled="!provider || !apiKey || selectedCount === 0">
          {{ t('models.wizard.finish') }}
        </Button>
      </DialogFooter>
    </DialogContent>
  </Dialog>
</template>
