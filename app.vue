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
  class: 'w-[500px]'
}, {
  key: 'sentimento',
  label: 'Sentimento',
  sortable: false,
  class: 'w-[100px]'
}]

const selected = ref([])
const page = ref(1)
const pageCount = 5
const apiUrl = 'http://localhost:5000/review'
const showLoading = ref(false)
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
  form.value.clear()
  showLoading.value = true
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
    refresh()
  }
  showLoading.value = false
}

async function deleteReviews() {
  selected.value.forEach(async (review) => {
    showLoading.value = true
    const res = await $fetch(`${apiUrl}?id=${review.id}`, { method: 'DELETE' })
    if (!res.error) {
      selected.value.pop()
      refresh()
    } else {
      form.value.setErrors([{
        // Map validation errors to { path: string, message: string }
        message: res.error,
        path: 'texto',
      }])
    }
    showLoading.value = false
  })
}

</script>
<template>
  <UContainer class="grid place-content-center h-screen">

    <UCard class="w-[800px]">

      <template #header>
        <h2 class="text-xl font-bold">Análise de sentimentos</h2>
      </template>

      <UForm ref="form" :schema="schema" :state="state" class="space-y-4" @submit="addReview">
        <UFormGroup name="modelo">
          <USelect placeholder="Selecione o modelo..." v-model="state.modelo" :options="models" option-attribute="name"
            :loading="showLoading" />
        </UFormGroup>
        <UFormGroup name="texto">
          <UTextarea placeholder="Informe o texto para análise de sentimentos ..." v-model="state.texto" class="w-full"
            :rows="10" :loading="showLoading" />
        </UFormGroup>
        <UButton type="submit" :loading="showLoading">Analisar</UButton>
      </UForm>

      <UDivider label="Resultados" type="dashed" />

      <UTable v-model="selected" :rows="rows" :columns="columns" :loading="showLoading"
        :loading-state="{ icon: 'i-heroicons-arrow-path-20-solid', label: 'Carregando...' }"
        :empty-state="{ icon: 'i-heroicons-circle-stack-20-solid', label: 'Nenhum review encontrado.' }"
        :progress="{ color: 'primary', animation: 'carousel' }" :ui="{ td: { base: 'max-w-[0] truncate' } }">
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
        <UButton :disabled="selected.length === 0" @click="deleteReviews" :loading="showLoading">Deletar ({{
          selected.length }})
        </UButton>
        <UPagination :disabled="reviews?.length === 0" v-model="page" :page-count="pageCount"
          :total="Array.isArray(reviews.value) ? reviews.value.length : 0" :loading="showLoading" />
      </div>

    </UCard>

  </UContainer>
</template>
