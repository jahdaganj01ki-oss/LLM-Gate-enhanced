<script setup lang="ts">
import { useI18n } from 'vue-i18n'
import { ref, computed } from 'vue'
import { useModelConfigStore } from '@/stores/models'
import { Button } from '@/components/ui/button'
import ModelCard from '@/components/models/ModelCard.vue'
import ModelWizardDialog from '@/components/models/ModelWizardDialog.vue'
import { Plus, Search, Package, Sparkles } from 'lucide-vue-next'
import { nanoid } from 'nanoid'
import { InputGroup, InputGroupAddon, InputGroupInput } from '@/components/ui/input-group'
import { toast } from 'vue-sonner'

const { t } = useI18n()
const modelStore = useModelConfigStore()

// Search
const searchQuery = ref('')

// Filtered model list
const filteredModels = computed(() => {
  const list = [...modelStore.configs].reverse()

  if (!searchQuery.value.trim()) {
    return list
  }
  const query = searchQuery.value.toLowerCase()
  return list.filter(model =>
    model.id.toLowerCase().includes(query) ||
    model.model?.toLowerCase().includes(query) ||
    model.baseUrl?.toLowerCase().includes(query)
  )
})

// Highlight newly created models
const newlyCreatedIds = ref<Set<string>>(new Set())

// Add new model
const handleAddModel = () => {
  const id = nanoid(6)
  modelStore.addConfig({
    id,
    model: '',
    baseUrl: '',
    apiKey: ''
  })
  newlyCreatedIds.value = new Set([id])
  // Clear search to ensure new model is visible
  searchQuery.value = ''
}

// Copy model
const handleCopyModel = (id: string) => {
  const original = modelStore.getById(id)
  if (!original) return

  const newId = nanoid(6)
  modelStore.addConfig({
    ...original,
    id: newId
  })
  newlyCreatedIds.value = new Set([newId])
  searchQuery.value = ''
}

// Models created via the quick add wizard
const handleWizardAdded = (ids: string[]) => {
  newlyCreatedIds.value = new Set(ids)
  // Clear search to ensure new models are visible
  searchQuery.value = ''
}

// Delete model
const handleDeleteModel = (id: string) => {
  const model = modelStore.getById(id)
  if (!model) return

  const backup = { ...model }
  modelStore.removeConfig(id)

  toast.success(t('models.deleted'), {
    description: model.model || model.id,
    action: {
      label: t('common.undo'),
      onClick: () => {
        modelStore.addConfig(backup)
      },
    },
  })
}

</script>

<template>
  <div class="p-4 space-y-3 h-full overflow-y-auto">
    <!-- Header -->
    <div class="flex items-center justify-between">
      <div>
        <h1 class="text-2xl font-semibold flex items-center">
          <Package class="h-6 w-6 mr-2" />
          {{ t('models.title') }}
        </h1>
        <div class="text-sm text-muted-foreground mt-1">
          {{ t('common.modelCount', modelStore.count) }}
        </div>
      </div>
      <div class="flex gap-2">
        <ModelWizardDialog @added="handleWizardAdded" />
        <Button @click="handleAddModel">
          <Plus />
          {{ t('models.add') }}
        </Button>
      </div>
    </div>

    <!-- Search box -->
    <InputGroup>
      <InputGroupInput type="search" v-model="searchQuery" :placeholder="t('models.search')" />
      <InputGroupAddon>
        <Search />
      </InputGroupAddon>
      <InputGroupAddon :align="'inline-end'" v-if="searchQuery">
        {{ t('common.resultCount', filteredModels.length) }}
      </InputGroupAddon>
    </InputGroup>

    <!-- Models list -->
    <div v-if="filteredModels.length > 0" class="grid gap-4 lg:grid-cols-2 xl:grid-cols-3">
      <TransitionGroup name="list">

        <ModelCard v-for="model in filteredModels" :key="model.id" :model="model" :is-new="newlyCreatedIds.has(model.id)"
          @delete="handleDeleteModel" @copy="handleCopyModel" />
      </TransitionGroup>
    </div>

    <!-- Empty state -->
    <div v-else class="flex flex-col items-center justify-center py-12 text-center">
      <div class="text-muted-foreground space-y-3">
        <Sparkles class="h-12 w-12 mx-auto opacity-50" />
        <p class="text-lg font-medium">
          {{ searchQuery ? t('models.emptyFiltered') : t('models.empty') }}
        </p>
        <p v-if="!searchQuery" class="text-sm">
          <Button variant="link" @click="handleAddModel">
            {{ t('models.add') }}
          </Button>
        </p>
      </div>
    </div>
  </div>
</template>
