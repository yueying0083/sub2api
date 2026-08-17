<template>
  <section aria-labelledby="prompt-user-activity-title" class="py-6">
    <div class="flex flex-wrap items-start justify-between gap-3">
      <div>
        <h2 id="prompt-user-activity-title" class="text-base font-semibold text-gray-950 dark:text-white">{{ t('admin.promptAudit.activity.title') }}</h2>
        <p class="mt-1 text-sm text-gray-500 dark:text-dark-300">{{ t('admin.promptAudit.activity.description') }}</p>
      </div>
      <div class="flex gap-2">
        <button type="button" class="btn btn-secondary btn-sm" :disabled="loading" @click="$emit('refresh')">{{ t('admin.promptAudit.actions.refresh') }}</button>
        <button type="button" class="btn btn-primary btn-sm" :disabled="loading || !hasExportRange" @click="$emit('export')">{{ t('admin.promptAudit.activity.exportCsv') }}</button>
      </div>
    </div>

    <div class="mt-5 grid gap-3 sm:grid-cols-3">
      <article class="rounded-xl border border-gray-200 bg-gray-50/70 p-4 dark:border-dark-700 dark:bg-dark-900/50">
        <p class="text-xs text-gray-500 dark:text-dark-400">{{ t('admin.promptAudit.activity.totalUsers') }}</p>
        <p class="mt-1 text-2xl font-semibold text-gray-950 dark:text-white">{{ totalUsers }}</p>
      </article>
      <article class="rounded-xl border border-gray-200 bg-gray-50/70 p-4 dark:border-dark-700 dark:bg-dark-900/50">
        <p class="text-xs text-gray-500 dark:text-dark-400">{{ t('admin.promptAudit.activity.totalPrompts') }}</p>
        <p class="mt-1 text-2xl font-semibold text-gray-950 dark:text-white">{{ totalPrompts }}</p>
      </article>
      <article class="rounded-xl border border-gray-200 bg-gray-50/70 p-4 dark:border-dark-700 dark:bg-dark-900/50">
        <p class="text-xs text-gray-500 dark:text-dark-400">{{ t('admin.promptAudit.activity.activeDays') }}</p>
        <p class="mt-1 text-2xl font-semibold text-gray-950 dark:text-white">{{ activeDays }}</p>
      </article>
    </div>

    <form class="mt-5 grid gap-3 sm:grid-cols-2 lg:grid-cols-4" @submit.prevent="submitFilters">
      <label class="text-xs text-gray-600 dark:text-dark-200">
        <span>{{ t('admin.promptAudit.activity.keyword') }}</span>
        <input v-model="localFilters.keyword" class="input mt-1 w-full" type="search" :placeholder="t('admin.promptAudit.activity.keywordHint')" />
      </label>
      <label class="text-xs text-gray-600 dark:text-dark-200">
        <span>{{ t('admin.promptAudit.events.startAt') }}</span>
        <input v-model="localFilters.start_at" type="datetime-local" class="input mt-1 w-full" />
      </label>
      <label class="text-xs text-gray-600 dark:text-dark-200">
        <span>{{ t('admin.promptAudit.events.endAt') }}</span>
        <input v-model="localFilters.end_at" type="datetime-local" class="input mt-1 w-full" />
      </label>
      <div class="flex items-end gap-2">
        <button type="submit" class="btn btn-primary btn-sm">{{ t('common.search') }}</button>
        <button type="button" class="btn btn-ghost btn-sm" @click="resetFilters">{{ t('common.reset') }}</button>
      </div>
    </form>

    <div v-if="error" role="alert" class="mt-4 rounded-lg bg-red-50 px-4 py-3 text-sm text-red-700 dark:bg-red-950/30 dark:text-red-300">{{ error }}</div>
    <div class="mt-5 overflow-x-auto rounded-xl border border-gray-200 dark:border-dark-700/60">
      <table class="min-w-[980px] w-full text-left text-sm">
        <thead class="bg-gray-50 text-xs uppercase tracking-wide text-gray-500 dark:bg-dark-900/70 dark:text-dark-400">
          <tr>
            <th class="px-4 py-3 font-medium">{{ t('admin.promptAudit.activity.member') }}</th>
            <th class="px-4 py-3 font-medium">{{ t('admin.promptAudit.activity.promptCount') }}</th>
            <th class="px-4 py-3 font-medium">{{ t('admin.promptAudit.activity.memberActiveDays') }}</th>
            <th class="px-4 py-3 font-medium">{{ t('admin.promptAudit.activity.lastActive') }}</th>
            <th class="px-4 py-3 font-medium">{{ t('admin.promptAudit.activity.modelsAndGroups') }}</th>
            <th class="px-4 py-3 font-medium">{{ t('admin.promptAudit.activity.latestPrompt') }}</th>
            <th class="px-4 py-3 text-right font-medium">{{ t('admin.promptAudit.common.actions') }}</th>
          </tr>
        </thead>
        <tbody class="divide-y divide-gray-100 bg-white dark:divide-dark-700 dark:bg-transparent">
          <tr v-if="loading"><td colspan="7" class="px-4 py-12 text-center text-gray-500">{{ t('common.loading') }}</td></tr>
          <tr v-else-if="items.length === 0"><td colspan="7" class="px-4 py-12 text-center text-gray-500">{{ t('admin.promptAudit.activity.empty') }}</td></tr>
          <tr v-for="item in items" v-else :key="item.user_id" class="align-top hover:bg-gray-50/70 dark:hover:bg-dark-800/70">
            <td class="px-4 py-3">
              <p class="font-medium text-gray-900 dark:text-white">{{ item.username || `#${item.user_id}` }}</p>
              <p class="mt-0.5 text-xs text-gray-500">{{ item.user_email || `ID ${item.user_id}` }}</p>
            </td>
            <td class="px-4 py-3 font-semibold text-gray-900 dark:text-white">{{ item.prompt_count }}</td>
            <td class="px-4 py-3">{{ item.active_days }}</td>
            <td class="px-4 py-3 whitespace-nowrap">{{ formatDate(item.last_active_at) }}</td>
            <td class="px-4 py-3">
              <p class="max-w-48 truncate">{{ item.models.join(', ') || '—' }}</p>
              <p class="mt-1 max-w-48 truncate text-xs text-gray-500">{{ item.groups.join(', ') || '—' }}</p>
            </td>
            <td class="px-4 py-3"><p class="max-w-sm line-clamp-3 whitespace-pre-wrap break-words text-gray-600 dark:text-dark-300">{{ item.last_prompt_preview || '—' }}</p></td>
            <td class="px-4 py-3 text-right">
              <button type="button" class="btn btn-ghost btn-sm" @click="$emit('view-user', item.user_id)">{{ t('admin.promptAudit.activity.viewRecords') }}</button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <div class="mt-4 flex flex-wrap items-center justify-between gap-3 text-sm text-gray-500 dark:text-dark-300">
      <span>{{ t('admin.promptAudit.activity.pageSummary', { page, pages: Math.max(pages, 1), total: totalUsers }) }}</span>
      <div class="flex items-center gap-2">
        <select :value="pageSize" class="input py-1.5" @change="$emit('page-size', Number(($event.target as HTMLSelectElement).value))">
          <option :value="20">20</option><option :value="50">50</option><option :value="100">100</option>
        </select>
        <button type="button" class="btn btn-secondary btn-sm" :disabled="page <= 1" @click="$emit('page', page - 1)">‹</button>
        <button type="button" class="btn btn-secondary btn-sm" :disabled="page >= pages" @click="$emit('page', page + 1)">›</button>
      </div>
    </div>
  </section>
</template>

<script setup lang="ts">
import { computed, reactive, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import type { PromptUserActivity, PromptUserActivityFilters } from '../types'

const props = defineProps<{
  items: PromptUserActivity[]
  totalUsers: number
  totalPrompts: number
  activeDays: number
  page: number
  pageSize: number
  pages: number
  filters: PromptUserActivityFilters
  loading: boolean
  error: string
}>()
const emit = defineEmits<{
  (event: 'search', value: PromptUserActivityFilters): void
  (event: 'refresh'): void
  (event: 'export'): void
  (event: 'page', value: number): void
  (event: 'page-size', value: number): void
  (event: 'view-user', value: number): void
}>()
const { t, locale } = useI18n()
const localFilters = reactive<PromptUserActivityFilters>({ ...props.filters })
const hasExportRange = computed(() => {
  const start = new Date(localFilters.start_at).getTime()
  const end = new Date(localFilters.end_at).getTime()
  return Number.isFinite(start) && Number.isFinite(end) && start < end
})
watch(() => props.filters, (value) => Object.assign(localFilters, value), { deep: true })

function submitFilters() { emit('search', { ...localFilters }) }
function resetFilters() {
  Object.assign(localFilters, { keyword: '', start_at: '', end_at: '' })
  submitFilters()
}
function formatDate(value: string): string {
  if (!value) return '—'
  return new Intl.DateTimeFormat(locale.value, { dateStyle: 'medium', timeStyle: 'short' }).format(new Date(value))
}
</script>
