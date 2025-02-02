<!--
  Copyright (C) 2023 Nethesis S.r.l.
  SPDX-License-Identifier: GPL-3.0-or-later
-->

<script setup lang="ts">
import { NeTitle, NeTabs, NeEmptyState } from '@nethserver/vue-tailwind-lib'
import { onMounted, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import { useRoute, useRouter } from 'vue-router'
import DpiRules from '@/components/standalone/dpi/DpiRules.vue'

//// review

const { t } = useI18n()
const route = useRoute()
const router = useRouter()

const tabs = ref([
  { name: 'rules', label: t('standalone.dpi.rules') },
  { name: 'exceptions', label: t('standalone.dpi.exceptions') }
])

let currentTab = ref('')
let initiallySelectedTab = ref('')

onMounted(() => {
  const tab = route.query.tab as string

  if (!tab) {
    initiallySelectedTab.value = 'rules'
  } else {
    initiallySelectedTab.value = tab
  }
})

function changeTab(tabName: any) {
  currentTab.value = tabName
  router.push({ path: route.path, query: { tab: currentTab.value } })
}
</script>

<template>
  <div class="text-sm">
    <NeTitle>{{ t('standalone.dpi.title') }}</NeTitle>
    <NeTabs
      :tabs="tabs"
      :selected="initiallySelectedTab"
      :srTabsLabel="t('ne_tabs.tabs')"
      :srSelectTabLabel="t('ne_tabs.select_a_tab')"
      @selectTab="changeTab"
      class="mb-8"
    />
    <div>
      <template v-if="currentTab === 'rules'">
        <DpiRules />
      </template>
      <template v-else-if="currentTab === 'exceptions'">
        <NeEmptyState
          :title="t('common.page_under_construction')"
          :icon="['fas', 'traffic-cone']"
        />
      </template>
    </div>
  </div>
</template>
