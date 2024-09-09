<script setup lang='ts'>
import type { CSSProperties } from 'vue'
import { computed, ref, watch } from 'vue'
import { NButton, NLayoutSider, useDialog, NTooltip, NModal, NCard, NInput } from 'naive-ui'
import List from './List.vue'
import mjFooter from './mjFooter.vue'
import { useAppStore, useChatStore } from '@/store'
import { useBasicLayout } from '@/hooks/useBasicLayout'
import { PromptStore, SvgIcon } from '@/components/common'
import { t } from '@/locales'

const appStore = useAppStore()
const chatStore = useChatStore()

const dialog = useDialog()

const { isMobile } = useBasicLayout()
const show = ref(false)

const collapsed = computed(() => appStore.siderCollapsed)

function handleAdd() {
  handleShowModal();
  chatStore.addHistory({ title: 'New Chat', uuid: Date.now(), isEdit: false })
  if (isMobile.value)
    appStore.setSiderCollapsed(true)
}

function handleUpdateCollapsed() {
  appStore.setSiderCollapsed(!collapsed.value)
}

function handleClearAll() {
  dialog.warning({
    title: t('chat.deleteMessage'),
    content: t('chat.clearHistoryConfirm'),
    positiveText: t('common.yes'),
    negativeText: t('common.no'),
    onPositiveClick: () => {
      chatStore.clearHistory()
      if (isMobile.value)
        appStore.setSiderCollapsed(true)
    },
  })
}

const getMobileClass = computed<CSSProperties>(() => {
  if (isMobile.value) {
    return {
      position: 'fixed',
      zIndex: 50,
      height: '100%',
    }
  }
  return {}
})

const mobileSafeArea = computed(() => {
  if (isMobile.value) {
    return {
      paddingBottom: 'env(safe-area-inset-bottom)',
    }
  }
  return {}
})

watch(
  isMobile,
  (val) => {
    appStore.setSiderCollapsed(val)
  },
  {
    immediate: true,
    flush: 'post',
  },
)

const showInfo = ref(false)
const handleShowInfo = () => {
  showInfo.value = !showInfo.value
}

const showModal = ref(false)
const handleShowModal = () => {
  showModal.value = !showModal.value
}
</script>

<template>
  <NLayoutSider :collapsed="collapsed" :collapsed-width="0" :width="260"
    :show-trigger="isMobile ? false : 'arrow-circle'" collapse-mode="transform" bordered :style="getMobileClass"
    @update-collapsed="handleUpdateCollapsed" class="aa-sider-c-trigger">
    <div class="flex flex-col h-full" :style="mobileSafeArea">
      <header
        class="sticky top-0 left-0 right-0 z-30 border-b dark:border-neutral-800 bg-white/80 dark:bg-black/20 backdrop-blur">
        <div @click="handleShowInfo"
          class=" cursor-pointer hover:bg-black/10 px-4 relative flex items-center justify-between min-w-0 overflow-hidden h-14"
          data-tauri-drag-region>
          <div class="text-base font-bold">
            常规
          </div>
          <!-- 下拉图标 -->
          <SvgIcon v-if="!showInfo" class="text-xl" icon="ri:arrow-down-s-line" />
          <!-- 关闭图标 -->
          <SvgIcon v-else class="text-xl" icon="ri:close-line" />
        </div>

        <!-- 下拉菜单 -->
        <div v-if="showInfo"
          class="absolute top-14 left-0 right-0 z-30 bg-gray-200 dark:bg-black rounded backdrop-blur m-2">
          <div class="bg-opacity-10">
            <div
              class="p-2 hover:bg-blue-700/80 hover:text-white text-sm hover:rounded flex flex-row cursor-pointer justify-between m-2">
              编辑服务器个人资料
              <SvgIcon class="text-xl" icon="ri:pencil-fill" />
            </div>
          </div>
        </div>
      </header>

      <main class="flex flex-col flex-1 min-h-0">
        <div class="flex flex-row justify-between items-center p-4 text-gray-400">
          <span>文字频道</span>

          <n-tooltip trigger="hover">
            <template #trigger>
              <SvgIcon @click="handleAdd" class="text-xl cursor-pointer" icon="ri:add-line" />
            </template>
            添加频道
          </n-tooltip>

        </div>
        <div class="flex-1 min-h-0 mb-2 overflow-hidden">
          <List />
        </div>
        <!-- <div class="flex items-center p-4 space-x-4">
          <div class="flex-1">
            <NButton block @click="show = true">
              {{ $t('store.siderButton') }}
            </NButton>
          </div>
          <NButton @click="handleClearAll">
            <SvgIcon icon="ri:close-circle-line" />
          </NButton>
        </div> -->
      </main>
      <mjFooter />
    </div>
  </NLayoutSider>
  <template v-if="isMobile">
    <div v-show="!collapsed" class="fixed inset-0 z-40 w-full h-full bg-black/40" @click="handleUpdateCollapsed" />
  </template>
  <PromptStore v-model:visible="show" />


  <n-modal closable v-model:show="showModal">
    <n-card style="width: 460px" title="创建频道" :bordered="false" size="huge" role="dialog" aria-modal="true">
      <div class="flex flex-col space-y-1 text-gray-800 dark:text-gray-200">
        <span class="text-gray-400">频道类别</span>
        <div
          class="flex flex-row items-center justify-between bg-gray-100 dark:bg-black/20 p-2.5 rounded cursor-pointer dark:hover:bg-black/40">
          <span class="text-3xl">#</span>
          <div class="flex flex-col">
            <span>文字频道</span>
            <span>发送消息、图片、GIF、表情符号、观点和梗</span>
          </div>
          <div>
            <SvgIcon class="text-2xl cursor-pointer" icon="ri:checkbox-circle-line" />
          </div>
        </div>
      </div>

      <div class="flex flex-col space-y-1 text-gray-200 mt-3">
        <span class="text-gray-400">频道名称</span>
        <div class="flex flex-row">
          <n-input placeholder="新-频道" class="dark:bg-black/20!" size="large">
            <template #prefix>
              <span class="text-xl">#</span>
            </template>
          </n-input>
        </div>
      </div>
      <template #footer>
        <div class="flex flex-row justify-end space-x-2">
          <NButton @click="showModal = false" quaternary>取消</NButton>
          <NButton type="info" color="#5865f2">
            <span class="text-white">创建频道</span>
          </NButton>
        </div>
      </template>
    </n-card>
  </n-modal>
</template>
