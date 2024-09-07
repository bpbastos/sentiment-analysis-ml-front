<script setup lang="ts">
import { z } from 'zod'
import type { FormSubmitEvent } from '#ui/types'

// Define o schema de validação usando zod
const schema = z.object({
  modelo: z.enum(['model-et', 'pipeline-et', 'model-distilbert'], {
    message: 'Selecione um modelo para análise de sentimentos.'
  }),
  texto: z.string({ required_error: 'Informe o texto para análise de sentimentos.' }).min(5, 'Texto deve conter no mínimo 5 caracteres').max(250, 'Texto deve conter no máximo 250 caracteres.')
})

type Schema = z.output<typeof schema>

const state = reactive({
  modelo: undefined,
  texto: undefined
})

const models = [
  {
    name: 'ExtraTrees',
    value: 'model-et'
  },
  {
    name: 'Pipeline ExtraTrees + MaxAbsScaler',
    value: 'pipeline-et'
  },
  {
    name: 'DistilBERT (Deep Learning)',
    value: 'model-distilbert'
  }
]

// Columns
const columns = [{
  key: 'texto',
  label: 'Texto',
  sortable: false,
  class: 'w-9/12'
}, {
  key: 'sentimento',
  label: '',
  sortable: false,
  class: 'w-1/12'
}]


const config = useRuntimeConfig();
const selected = ref([])
const page = ref(1)
const pageCount = 5
const apiUrl = config.public.API_URL
const isAddingReview = ref(false)
const isDeletingReview = ref(false)
const form = ref()

const { status, data, refresh } = await useLazyAsyncData('reviews', () => $fetch(apiUrl))

const reviews = computed(() => {
  if (!Array.isArray(data.value)) return []
  return data.value
})

const rows = computed(() => {
  if (!Array.isArray(reviews.value)) return []
  return reviews.value.slice((page.value - 1) * pageCount, page.value * pageCount)
})

async function addReview(event: FormSubmitEvent<Schema>) {
  isAddingReview.value = true
  try {
    const formData = new FormData()
    formData.append('modelo', event.data.modelo)
    formData.append('texto', event.data.texto)
    const res = await $fetch(apiUrl, {
      method: 'POST',
      body: formData,
    })
    if (res.error) {
      form.value.setErrors([{
        // Map validation errors to { path: string, message: string }
        message: res.error,
        path: 'texto',
      }])
    } else {
      state.modelo = undefined
      state.texto = undefined    
      form.value.clear()
      await refresh()
    }
  } catch (error) {
    console.error('Erro ao adicionar review:', error)
    form.value.setErrors([{
      message: 'Erro ao adicionar review. Tente novamente mais tarde.',
      path: 'texto',
    }])
  } finally {
    isAddingReview.value = false
  }
}

async function deleteReviews() {
  isDeletingReview.value = true
  try {
    const reviewsToDelete = [...selected.value] // Cria uma cópia da lista de reviews selecionados
    for (const review of reviewsToDelete) {
      const res = await $fetch(`${apiUrl}?id=${review.id}`, { method: 'DELETE' })
      if (res.error) {
        form.value.setErrors([{
          message: res.error,
          path: 'texto',
        }])
      } else {
        // Remove o review deletado da lista de selecionados
        const index = selected.value.findIndex(r => r.id === review.id)
        if (index !== -1) {
          selected.value.splice(index, 1)
        }
        // Atualiza a lista de reviews
        await refresh()
      }
    }
  } catch (error) {
    console.error('Erro ao deletar reviews:', error)
    form.value.setErrors([{
      message: 'Erro ao deletar reviews. Tente novamente mais tarde.',
      path: 'texto',
    }])
  } finally {
    isDeletingReview.value = false
  }
}
</script>
<template>
  <UContainer class="grid place-content-center h-screen">

    <UCard class="w-[800px]">

      <template #header>
        <h2 class="text-xl font-bold">Análise de sentimentos</h2>
      </template>

      <UForm ref="form" :schema="schema" :state="state" class="space-y-4" @submit="addReview" :loading="isAddingReview">
        <UFormGroup name="modelo">
          <USelect placeholder="Selecione o modelo..." v-model="state.modelo" :options="models"
            option-attribute="name" />
        </UFormGroup>
        <UFormGroup name="texto">
          <UTextarea placeholder="Informe o texto para análise de sentimentos ..." v-model="state.texto" class="w-full"
            :rows="10" />
        </UFormGroup>
        <UButton type="submit" :loading="isAddingReview">Analisar</UButton>
      </UForm>

      <UDivider label="Resultados" type="dashed" />

      <UTable v-model="selected" :rows="rows" :columns="columns" class="w-full"
        :loading="status === 'pending' || isAddingReview || isDeletingReview"
        :loading-state="{ icon: 'i-heroicons-arrow-path-20-solid', label: 'Carregando...' }"
        :progress="{ color: 'primary', animation: 'carousel' }" :ui="{ td: { base: 'max-w-[0] truncate' } }">
        <template #empty-state>
  
              <span class="italic text-sm">Nenhum review encontrado.</span>
         
        </template>
        <template #sentimento-data="{ row }">
          <div v-show="row.sentimento" class="text-3xl">😊</div>
          <div v-show="!row.sentimento" class="text-3xl">🙁</div>
        </template>
        <template #expand="{ row }">
          <UCard>
            <template #header>
              {{ row.id }}
            </template>
            "{{ row.texto }}" - {{ row.sentimento ? '🙂' : '🙁' }}
            <template #footer>
              "{{ models.find(model => model.value === row.modelo)?.name }}" em {{ row.data_criacao }}
            </template>
          </UCard>
        </template>
      </UTable>

      <div class="flex justify-between px-3 py-3.5 border-t border-gray-200 dark:border-gray-700">
        <div v-if="Array.isArray(reviews) && reviews.length > 0">
          <UButton :disabled="selected.length === 0" @click="deleteReviews" :loading="isDeletingReview">
            Deletar <span v-if="selected.length > 0">({{ selected.length }})</span>
          </UButton>
        </div>
        <div v-if="Array.isArray(reviews) && reviews.length > 1" class="ml-auto">
          <UPagination v-model="page" :page-count="pageCount" :total="reviews.length" />
        </div>
      </div>

    </UCard>

  </UContainer>
</template>
