<script setup lang="ts">
import { inject } from '@vue/runtime-core'
import {
  Empty,
  ProjectIdInj,
  computed,
  extractSdkResponseErrorMsg,
  h,
  iconMap,
  message,
  onMounted,
  storeToRefs,
  useBase,
  useGlobal,
  useI18n,
  useNuxtApp,
} from '#imports'

const props = defineProps<{
  sourceId: string
}>()

const { t } = useI18n()

const { $api, $e } = useNuxtApp()

const { base } = storeToRefs(useBase())

const _projectId = inject(ProjectIdInj, ref())
const baseId = computed(() => _projectId.value ?? base.value?.id)

const { includeM2M } = useGlobal()

const roles = ref<string[]>(['editor', 'commenter', 'viewer'])

const isLoading = ref(false)

const tables = ref<any[]>([])

const searchInput = ref('')

const filteredTables = computed(() =>
  tables.value.filter(
    (el) =>
      el?.source_id === props.sourceId &&
      ((typeof el?._ptn === 'string' && el._ptn.toLowerCase().includes(searchInput.value.toLowerCase())) ||
        (typeof el?.title === 'string' && el.title.toLowerCase().includes(searchInput.value.toLowerCase()))),
  ),
)

async function loadTableList() {
  try {
    if (!baseId.value) return

    isLoading.value = true

    tables.value = await $api.base.modelVisibilityList(baseId.value, {
      includeM2M: includeM2M.value,
    })
  } catch (e) {
    console.error(e)
  } finally {
    isLoading.value = false
  }
}

async function saveUIAcl() {
  try {
    if (!baseId.value) return

    await $api.base.modelVisibilitySet(
      baseId.value,
      tables.value.filter((t) => t.edited),
    )
    // Updated UI ACL for tables successfully
    message.success(t('msg.success.updatedUIACL'))
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }
  $e('a:proj-meta:ui-acl')
}

const onRoleCheck = (record: any, role: string) => {
  record.disabled[role] = !record.disabled[role]
  record.edited = true
}

onMounted(async () => {
  if (tables.value.length === 0) {
    await loadTableList()
  }
})

const tableHeaderRenderer = (label: string) => () => h('div', { class: 'text-gray-500' }, label)

const columns = [
  {
    title: tableHeaderRenderer(t('labels.tableName')),
    name: 'table_name',
  },
  {
    title: tableHeaderRenderer(t('labels.viewName')),
    name: 'view_name',
  },
  {
    title: tableHeaderRenderer(t('objects.roleType.editor')),
    name: 'editor',
    width: 120,
  },
  {
    title: tableHeaderRenderer(t('objects.roleType.commenter')),
    name: 'commenter',
    width: 120,
  },
  {
    title: tableHeaderRenderer(t('objects.roleType.viewer')),
    name: 'viewer',
    width: 120,
  },
]
</script>

<template>
  <div class="flex flex-row w-full items-center justify-center">
    <div class="flex flex-col w-[900px]">
      <span class="mb-4 first-letter:capital font-bold"> UI ACL : {{ base.title }} </span>
      <div class="flex flex-row items-center w-full mb-4 gap-2 justify-between">
        <a-input v-model:value="searchInput" :placeholder="$t('placeholder.searchModels')" class="nc-acl-search !w-[400px]">
          <template #prefix>
            <component :is="iconMap.search" />
          </template>
        </a-input>
        <div class="flex">
          <a-button type="text" ghost class="self-start !rounded-md nc-acl-reload" @click="loadTableList">
            <div class="flex items-center gap-2 text-gray-600 font-light">
              <component :is="iconMap.reload" :class="{ 'animate-infinite animate-spin !text-success': isLoading }" />
              {{ $t('general.reload') }}
            </div>
          </a-button>

          <NcButton size="large" class="z-10 !rounded-lg !px-2 mr-2.5" type="primary" @click="saveUIAcl">
            <div class="flex flex-row items-center w-full gap-x-1">
              <component :is="iconMap.save" />
              <div class="flex">{{ $t('general.save') }}</div>
            </div>
          </NcButton>
        </div>
      </div>

      <div class="max-h-600px overflow-y-auto">
        <a-table
          class="w-full"
          size="small"
          :data-source="filteredTables"
          :columns="columns"
          :pagination="false"
          :loading="isLoading"
          sticky
          bordered
          :custom-row="
            (record) => ({
              class: `nc-acl-table-row nc-acl-table-row-${record.title}`,
            })
          "
        >
          <template #emptyText>
            <a-empty :image="Empty.PRESENTED_IMAGE_SIMPLE" :description="$t('labels.noData')" />
          </template>

          <template #bodyCell="{ record, column }">
            <div v-if="column.name === 'table_name'">
              <div class="flex items-center gap-1">
                <div class="min-w-5 flex items-center justify-center">
                  <GeneralTableIcon :meta="{ meta: record.table_meta, type: record.ptype }" class="text-gray-500" />
                </div>
                <GeneralTruncateText>
                  <span class="overflow-ellipsis min-w-0 shrink-1">{{ record._ptn }}</span>
                </GeneralTruncateText>
              </div>
            </div>

            <div v-if="column.name === 'view_name'">
              <div class="flex items-center gap-1">
                <div class="min-w-5 flex items-center justify-center">
                  <GeneralViewIcon :meta="record" class="text-gray-500"></GeneralViewIcon>
                </div>
                <span class="overflow-ellipsis min-w-0 shrink-1">{{ record.title }}</span>
              </div>
            </div>

            <div v-for="role in roles" :key="role">
              <div v-if="column.name === role">
                <a-tooltip>
                  <template #title>
                    <span v-if="record.disabled[role]">
                      {{ $t('labels.clickToMake') }} '{{ record.title }}' {{ $t('labels.visibleForRole') }} {{ role }}
                      {{ $t('labels.inUI') }} dashboard</span
                    >
                    <span v-else
                      >{{ $t('labels.clickToHide') }}'{{ record.title }}' {{ $t('labels.forRole') }}:{{ role }}
                      {{ $t('labels.inUI') }}</span
                    >
                  </template>

                  <a-checkbox
                    :checked="!record.disabled[role]"
                    :class="`nc-acl-${record.title}-${role}-chkbox`"
                    @change="onRoleCheck(record, role)"
                  />
                </a-tooltip>
              </div>
            </div>
          </template>
        </a-table>
      </div>
    </div>
  </div>
</template>
