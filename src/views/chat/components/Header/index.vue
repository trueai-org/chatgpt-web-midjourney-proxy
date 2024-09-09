<script lang="ts" setup>
import { computed, nextTick, ref, watch } from 'vue'
import { HoverButton, SvgIcon } from '@/components/common'
import { gptConfigStore, homeStore, useAppStore, useChatStore } from '@/store'
import { useBasicLayout } from '@/hooks/useBasicLayout'
import { NModal, NInputGroup, NInput, NButton } from "naive-ui"
import aiModel from "@/views/mj/aiModel.vue"
import { chatSetting, mlog } from '@/api'
import { debounce } from '@/utils/functions/debounce'
import { GPTUserAvatar } from '@/components/common'

const { isMobile } = useBasicLayout()

interface Props {
  usingContext: boolean
}

interface Emit {
  (ev: 'export'): void
  (ev: 'handleClear'): void
  (ev: 'handleHide'): void
}

defineProps<Props>()

const emit = defineEmits<Emit>()

const appStore = useAppStore()
const chatStore = useChatStore()

const collapsed = computed(() => appStore.siderCollapsed)
const currentChatHistory = computed(() => chatStore.getChatHistoryByCurrentActive)

function handleUpdateCollapsed() {
  appStore.setSiderCollapsed(!collapsed.value)
}

function onScrollToTop() {
  const scrollRef = document.querySelector('#scrollRef')
  if (scrollRef)
    nextTick(() => scrollRef.scrollTop = 0)
}

function handleExport() {
  emit('export')
}

function handleClear() {
  emit('handleClear')
}

function handleHide() {
  emit('handleHide')
}

const uuid = chatStore.active;
const chatSet = new chatSetting(uuid == null ? 1002 : uuid);
const nGptStore = ref();
nGptStore.value = chatSet.getGptConfig();
const st = ref({ isShow: false });
//导致卡死的原因 当删除时触发 切换 uuid 这个地方会删除的uuid 跟新uuid 一直却换
watch(() => gptConfigStore.myData, debounce(() => {
  mlog("toMyuid19", "watch gptConfigStore.myData ", chatStore.active)
  nGptStore.value = chatSet.getGptConfig()
}, 600), { deep: true })
watch(() => homeStore.myData.act, debounce((n) => n == 'saveChat' && (nGptStore.value = chatSet.getGptConfig()), 600), { deep: true })
</script>

<template>
  <header class="sticky top-0 left-0 right-0 z-30 dark:border-neutral-800 bg-white/80 dark:bg-black/20 backdrop-blur"
    :class="homeStore.myData.local == 'draw' ? 'border-b' : ''">

    <div v-if="homeStore.myData.local == 'draw'"
      class="relative flex items-center justify-between min-w-0 overflow-hidden h-14" data-tauri-drag-region>
      <div class="flex items-center">
        <button class="flex items-center justify-center w-11 h-11" @click="handleUpdateCollapsed" v-if="isMobile">
          <SvgIcon v-if="collapsed" class="text-2xl" icon="ri:align-justify" />
          <SvgIcon v-else class="text-2xl" icon="ri:align-right" />
        </button>
      </div>
      <h1
        class="flex-1 px-4 pr-6 overflow-hidden cursor-pointer select-none text-ellipsis whitespace-nowrap items-center flex"
        @dblclick="onScrollToTop" data-tauri-drag-region>
        <span class="text-3xl mr-2 text-gray-400">#</span>
        {{ currentChatHistory?.title ?? '' }}
      </h1>
      <div class="flex items-center space-x-2 mr-2">
        <HoverButton @click="handleExport">
          <span class="text-xl text-[#4f555e] dark:text-white">
            <SvgIcon icon="ri:download-2-line" />
          </span>
        </HoverButton>
        <HoverButton @click="handleClear">
          <span class="text-xl text-[#4f555e] dark:text-white">
            <SvgIcon icon="ri:delete-bin-line" />
          </span>
        </HoverButton>

        <HoverButton @click="handleHide">
          <span class="text-xl text-[#4f555e] dark:text-white">
            <svg x="0" y="0" class="icon_fc4f04" aria-hidden="true" role="img" xmlns="http://www.w3.org/2000/svg"
              width="18" height="18" fill="none" viewBox="0 0 24 24">
              <path fill="currentColor"
                d="M14.5 8a3 3 0 1 0-2.7-4.3c-.2.4.06.86.44 1.12a5 5 0 0 1 2.14 3.08c.01.06.06.1.12.1ZM18.44 17.27c.15.43.54.73 1 .73h1.06c.83 0 1.5-.67 1.5-1.5a7.5 7.5 0 0 0-6.5-7.43c-.55-.08-.99.38-1.1.92-.06.3-.15.6-.26.87-.23.58-.05 1.3.47 1.63a9.53 9.53 0 0 1 3.83 4.78ZM12.5 9a3 3 0 1 1-6 0 3 3 0 0 1 6 0ZM2 20.5a7.5 7.5 0 0 1 15 0c0 .83-.67 1.5-1.5 1.5a.2.2 0 0 1-.2-.16c-.2-.96-.56-1.87-.88-2.54-.1-.23-.42-.15-.42.1v2.1a.5.5 0 0 1-.5.5h-8a.5.5 0 0 1-.5-.5v-2.1c0-.25-.31-.33-.42-.1-.32.67-.67 1.58-.88 2.54a.2.2 0 0 1-.2.16A1.5 1.5 0 0 1 2 20.5Z"
                class=""></path>
            </svg>
          </span>
        </HoverButton>

        <n-input size="small" placeholder="搜索">
          <template #suffix>
            <SvgIcon icon="ri:search-line" />
          </template>
        </n-input>

      </div>
    </div>

    <div v-if="homeStore.myData.local != 'draw'"
      class="relative flex items-center justify-between min-w-0 overflow-hidden h-14" data-tauri-drag-region>

      <div @click="st.isShow = true"
        class="hover:bg-gray-100 text-xl font-bold text-gray-500 py-1 px-1 mx-2 cursor-pointer  rounded  bg-white dark:border-neutral-800 dark:bg-[#111114]">
        <div class="flex items-center justify-center space-x-1 cursor-pointer hover:text-[#4b9e5f]"
          v-if="homeStore.myData.local != 'draw'">
          <template v-if="nGptStore.gpts">
            <!-- <SvgIcon icon="ri:apps-fill" /> -->
            <span class="line-clamp-1 overflow-hidden">{{ nGptStore.gpts.name }}</span>
          </template>
          <template v-else>
            <!-- <SvgIcon icon="heroicons:sparkles" /> -->
            <span>{{ nGptStore.model }}</span>
          </template>
          <SvgIcon icon="ri:arrow-down-s-line" class="text-xl text-gray-400" />
        </div>
      </div>

      <div class="flex items-center space-x-3 mr-2">
        <HoverButton @click="handleExport">
          <span class="text-xl text-[#4f555e] dark:text-white">
            <SvgIcon icon="ri:share-2-line" />
          </span>
        </HoverButton>
        <!-- <HoverButton @click="handleClear">
          <span class="text-xl text-[#4f555e] dark:text-white">
            <SvgIcon icon="ri:delete-bin-line" />
          </span>
        </HoverButton> -->

        <!-- <HoverButton @click="handleHide">
          <span class="text-xl text-[#4f555e] dark:text-white">
            <svg x="0" y="0" class="icon_fc4f04" aria-hidden="true" role="img" xmlns="http://www.w3.org/2000/svg"
              width="18" height="18" fill="none" viewBox="0 0 24 24">
              <path fill="currentColor"
                d="M14.5 8a3 3 0 1 0-2.7-4.3c-.2.4.06.86.44 1.12a5 5 0 0 1 2.14 3.08c.01.06.06.1.12.1ZM18.44 17.27c.15.43.54.73 1 .73h1.06c.83 0 1.5-.67 1.5-1.5a7.5 7.5 0 0 0-6.5-7.43c-.55-.08-.99.38-1.1.92-.06.3-.15.6-.26.87-.23.58-.05 1.3.47 1.63a9.53 9.53 0 0 1 3.83 4.78ZM12.5 9a3 3 0 1 1-6 0 3 3 0 0 1 6 0ZM2 20.5a7.5 7.5 0 0 1 15 0c0 .83-.67 1.5-1.5 1.5a.2.2 0 0 1-.2-.16c-.2-.96-.56-1.87-.88-2.54-.1-.23-.42-.15-.42.1v2.1a.5.5 0 0 1-.5.5h-8a.5.5 0 0 1-.5-.5v-2.1c0-.25-.31-.33-.42-.1-.32.67-.67 1.58-.88 2.54a.2.2 0 0 1-.2.16A1.5 1.5 0 0 1 2 20.5Z"
                class=""></path>
            </svg>
          </span>
        </HoverButton> -->

        <!-- <n-input size="small" placeholder="搜索">
          <template #suffix>
            <SvgIcon icon="ri:search-line" />
          </template>
        </n-input> -->
        <GPTUserAvatar />

        
      </div>
    </div>

    <!-- <div @click="st.isShow = true"
      class="hover:bg-gray-100 font-bold text-gray-500 py-1 px-1 mx-2 absolute left-0 top-full cursor-pointer  rounded  bg-white dark:border-neutral-800 dark:bg-[#111114] mt-4">
      <div class="flex items-center justify-center space-x-1 cursor-pointer hover:text-[#4b9e5f]"
        v-if="homeStore.myData.local != 'draw'">
        <template v-if="nGptStore.gpts">
          <SvgIcon icon="ri:apps-fill" />
          <span class="line-clamp-1 overflow-hidden">{{ nGptStore.gpts.name }}</span>
        </template>
        <template v-else>
          <SvgIcon icon="heroicons:sparkles" />
          <span>{{ nGptStore.model }}</span>
        </template>
        <SvgIcon icon="ri:arrow-down-s-line" class="text-xl text-gray-400" />
      </div>
    </div> -->
  </header>

  <NModal v-model:show="st.isShow" preset="card" :title="$t('mjchat.modelChange')" class="!max-w-[620px]"
    @close="st.isShow = false">
    <aiModel @close="st.isShow = false" />
  </NModal>
</template>
