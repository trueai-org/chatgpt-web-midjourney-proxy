<script setup lang='ts'>
import type { Ref } from 'vue'
import { computed, h, nextTick, onMounted, onUnmounted, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { storeToRefs } from 'pinia'
import {
  NAutoComplete,
  NAvatar,
  NButton,
  NInput,
  useDialog,
  useMessage,
  NPopover
} from 'naive-ui'
import html2canvas from 'html2canvas'
import drawListVue from '../mj/drawList.vue'
import aiGPT from '../mj/aiGpt.vue'
import AiSiderInput from '../mj/aiSiderInput.vue'
import aiGptInput from '../mj/aiGptInput.vue'
import mjSiderPeople from '../mj/mjSiderPeople.vue'
import AiTextSetting from '../mj/aiTextSetting.vue'

import { Message } from './components'

import { useScroll } from './hooks/useScroll'
import { useChat } from './hooks/useChat'
import { useUsingContext } from './hooks/useUsingContext'
import HeaderComponent from './components/Header/index.vue'

import { SvgIcon } from '@/components/common'
import { useBasicLayout } from '@/hooks/useBasicLayout'
import {
  gptConfigStore,
  gptServerStore,
  gptsUlistStore,
  homeStore,
  useChatStore,
  usePromptStore,
  useUserStore,
} from '@/store'
import type { gptsType } from '@/api'
import {
  chatSetting,
  fetchChatAPIProcess,
  mlog,
  myFetch,
} from '@/api'
import { t } from '@/locales'


const json = {
  "mj_commands": [
    {
      "type": 1,
      "name": "imagine",
      "description": "Create images with Midjourney"
    },
    {
      "type": 1,
      "name": "info",
      "description": "View information about your profile."
    },
    {
      "type": 1,
      "name": "fast",
      "description": "Switch to fast mode"
    },
    {
      "type": 1,
      "name": "relax",
      "description": "Switch to relax mode"
    },
    {
      "type": 1,
      "name": "settings",
      "description": "View and adjust your personal settings."
    },
    {
      "type": 1,
      "name": "blend",
      "description": "Blend images together seamlessly!"
    },
    {
      "type": 1,
      "name": "describe",
      "description": "Writes a prompt based on your image."
    },
    {
      "type": 1,
      "name": "shorten",
      "description": "Analyzes and shortens a prompt."
    },
    {
      "type": 1,
      "name": "show",
      "description": "Shows the job view based on job id."
    }
  ],
  "niji_commands": [
    {
      "type": 1,
      "name": "imagine",
      "description": "Create images with Midjourney"
    },
    {
      "type": 1,
      "name": "info",
      "description": "View information about your profile."
    },
    {
      "type": 1,
      "name": "fast",
      "description": "Switch to fast mode"
    },
    {
      "type": 1,
      "name": "relax",
      "description": "Switch to relax mode"
    },
    {
      "type": 1,
      "name": "settings",
      "description": "View and adjust your personal settings."
    },
    {
      "type": 1,
      "name": "blend",
      "description": "Blend images together seamlessly!"
    },
    {
      "type": 1,
      "name": "describe",
      "description": "Writes a prompt based on your image."
    },
    {
      "type": 1,
      "name": "shorten",
      "description": "Analyzes and shortens a prompt."
    },
    {
      "type": 1,
      "name": "show",
      "description": "Shows the job view based on job id."
    }
  ]
};

let controller = new AbortController()

const openLongReply = import.meta.env.VITE_GLOB_OPEN_LONG_REPLY === 'true'

const route = useRoute()
const dialog = useDialog()
const ms = useMessage()
const router = useRouter()
const chatStore = useChatStore()

const { isMobile } = useBasicLayout()
const { addChat, updateChat, updateChatSome, getChatByUuidAndIndex }
  = useChat()
const { scrollRef, scrollToBottom, scrollToBottomIfAtBottom } = useScroll()
const { usingContext, toggleUsingContext } = useUsingContext()

const { uuid } = route.params as { uuid: string }

const dataSources = computed(() => chatStore.getChatByUuid(+uuid))
const conversationList = computed(() =>
  dataSources.value.filter(
    item => !item.inversion && !!item.conversationOptions,
  ),
)

const prompt = ref<string>('')
const loading = ref<boolean>(false)
const inputRef = ref<Ref | null>(null)

// 添加PromptStore
const promptStore = usePromptStore()

// 使用storeToRefs，保证store修改后，联想部分能够重新渲染
const { promptList: promptTemplate } = storeToRefs<any>(promptStore)

// 未知原因刷新页面，loading 状态不会重置，手动重置
dataSources.value.forEach((item, index) => {
  if (item.loading)
    updateChatSome(+uuid, index, { loading: false })
})

const userStore = useUserStore()

const userInfo = computed(() => userStore.userInfo)

const backgroundImage = computed(() => userInfo.value.backgroundImage ?? '')

function handleSubmit() {
  // onConversation() //把这个放到aiGpt
  const message = prompt.value
  if (!message || message.trim() === '')
    return
  if (loading.value)
    return
  loading.value = true
  homeStore.setMyData({
    act: 'gpt.submit',
    actData: { prompt: prompt.value, uuid },
  })
  prompt.value = ''
}

async function onConversation() {
  let message = prompt.value

  if (loading.value)
    return

  if (!message || message.trim() === '')
    return

  controller = new AbortController()

  addChat(+uuid, {
    dateTime: new Date().toLocaleString(),
    text: message,
    inversion: true,
    error: false,
    conversationOptions: null,
    requestOptions: { prompt: message, options: null },
  })
  scrollToBottom()

  loading.value = true
  prompt.value = ''

  let options: Chat.ConversationRequest = {}
  const lastContext
    = conversationList.value[conversationList.value.length - 1]
      ?.conversationOptions

  if (lastContext && usingContext.value)
    options = { ...lastContext }

  addChat(+uuid, {
    dateTime: new Date().toLocaleString(),
    text: '思考中',
    loading: true,
    inversion: false,
    error: false,
    conversationOptions: null,
    requestOptions: { prompt: message, options: { ...options } },
  })
  scrollToBottom()

  try {
    let lastText = ''
    const fetchChatAPIOnce = async () => {
      await fetchChatAPIProcess<Chat.ConversationResponse>({
        prompt: message,
        options,
        signal: controller.signal,
        onDownloadProgress: ({ event }) => {
          const xhr = event.target
          const { responseText } = xhr
          // Always process the final line
          const lastIndex = responseText.lastIndexOf(
            '\n',
            responseText.length - 2,
          )
          let chunk = responseText
          if (lastIndex !== -1)
            chunk = responseText.substring(lastIndex)
          try {
            const data = JSON.parse(chunk)
            updateChat(+uuid, dataSources.value.length - 1, {
              dateTime: new Date().toLocaleString(),
              text: lastText + (data.text ?? ''),
              inversion: false,
              error: false,
              loading: true,
              conversationOptions: {
                conversationId: data.conversationId,
                parentMessageId: data.id,
              },
              requestOptions: { prompt: message, options: { ...options } },
            })

            if (
              openLongReply
              && data.detail.choices[0].finish_reason === 'length'
            ) {
              options.parentMessageId = data.id
              lastText = data.text
              message = ''
              return fetchChatAPIOnce()
            }

            scrollToBottomIfAtBottom()
          }
          catch (error) {
            //
          }
        },
      })
      updateChatSome(+uuid, dataSources.value.length - 1, { loading: false })
    }

    await fetchChatAPIOnce()
  }
  catch (error: any) {
    const errorMessage = error?.message ?? t('common.wrong')

    if (error.message === 'canceled') {
      updateChatSome(+uuid, dataSources.value.length - 1, {
        loading: false,
      })
      scrollToBottomIfAtBottom()
      return
    }

    const currentChat = getChatByUuidAndIndex(
      +uuid,
      dataSources.value.length - 1,
    )

    if (currentChat?.text && currentChat.text !== '') {
      updateChatSome(+uuid, dataSources.value.length - 1, {
        text: `${currentChat.text}\n[${errorMessage}]`,
        error: false,
        loading: false,
      })
      return
    }

    updateChat(+uuid, dataSources.value.length - 1, {
      dateTime: new Date().toLocaleString(),
      text: errorMessage,
      inversion: false,
      error: true,
      loading: false,
      conversationOptions: null,
      requestOptions: { prompt: message, options: { ...options } },
    })
    scrollToBottomIfAtBottom()
  }
  finally {
    loading.value = false
  }
}

async function onRegenerate(index: number) {
  if (loading.value)
    return

  controller = new AbortController()

  const { requestOptions } = dataSources.value[index]

  let message = requestOptions?.prompt ?? ''

  let options: Chat.ConversationRequest = {}

  if (requestOptions.options)
    options = { ...requestOptions.options }

  loading.value = true

  updateChat(+uuid, index, {
    dateTime: new Date().toLocaleString(),
    text: '',
    inversion: false,
    error: false,
    loading: true,
    conversationOptions: null,
    requestOptions: { prompt: message, options: { ...options } },
  })

  try {
    let lastText = ''
    const fetchChatAPIOnce = async () => {
      await fetchChatAPIProcess<Chat.ConversationResponse>({
        prompt: message,
        options,
        signal: controller.signal,
        onDownloadProgress: ({ event }) => {
          const xhr = event.target
          const { responseText } = xhr
          // Always process the final line
          const lastIndex = responseText.lastIndexOf(
            '\n',
            responseText.length - 2,
          )
          let chunk = responseText
          if (lastIndex !== -1)
            chunk = responseText.substring(lastIndex)
          try {
            const data = JSON.parse(chunk)
            updateChat(+uuid, index, {
              dateTime: new Date().toLocaleString(),
              text: lastText + (data.text ?? ''),
              inversion: false,
              error: false,
              loading: true,
              conversationOptions: {
                conversationId: data.conversationId,
                parentMessageId: data.id,
              },
              requestOptions: { prompt: message, options: { ...options } },
            })

            if (
              openLongReply
              && data.detail.choices[0].finish_reason === 'length'
            ) {
              options.parentMessageId = data.id
              lastText = data.text
              message = ''
              return fetchChatAPIOnce()
            }
          }
          catch (error) {
            //
          }
        },
      })
      updateChatSome(+uuid, index, { loading: false })
    }
    await fetchChatAPIOnce()
  }
  catch (error: any) {
    if (error.message === 'canceled') {
      updateChatSome(+uuid, index, {
        loading: false,
      })
      return
    }

    const errorMessage = error?.message ?? t('common.wrong')

    updateChat(+uuid, index, {
      dateTime: new Date().toLocaleString(),
      text: errorMessage,
      inversion: false,
      error: true,
      loading: false,
      conversationOptions: null,
      requestOptions: { prompt: message, options: { ...options } },
    })
  }
  finally {
    loading.value = false
  }
}

function handleExport() {
  if (loading.value)
    return

  const d = dialog.warning({
    title: t('chat.exportImage'),
    content: t('chat.exportImageConfirm'),
    positiveText: t('common.yes'),
    negativeText: t('common.no'),
    onPositiveClick: async () => {
      try {
        d.loading = true
        const ele = document.getElementById('image-wrapper')
        const canvas = await html2canvas(ele as HTMLDivElement, {
          useCORS: true,
        })
        const imgUrl = canvas.toDataURL('image/png')
        const tempLink = document.createElement('a')
        tempLink.style.display = 'none'
        tempLink.href = imgUrl
        tempLink.setAttribute('download', 'chat-shot.png')
        if (typeof tempLink.download === 'undefined')
          tempLink.setAttribute('target', '_blank')

        document.body.appendChild(tempLink)
        tempLink.click()
        document.body.removeChild(tempLink)
        window.URL.revokeObjectURL(imgUrl)
        d.loading = false
        ms.success(t('chat.exportSuccess'))
        Promise.resolve()
      }
      catch (error: any) {
        ms.error(t('chat.exportFailed'))
      }
      finally {
        d.loading = false
      }
    },
  })
}

function handleDelete(index: number) {
  if (loading.value)
    return

  dialog.warning({
    title: t('chat.deleteMessage'),
    content: t('chat.deleteMessageConfirm'),
    positiveText: t('common.yes'),
    negativeText: t('common.no'),
    onPositiveClick: () => {
      chatStore.deleteChatByUuid(+uuid, index)
    },
  })
}

function handleEdit(index: number) {
  if (loading.value)
    return

  const editedMessage = ref(dataSources.value[index].text.slice())

  const inputNode = () => {
    return h(NInput, {
      'value': editedMessage.value,
      'type': 'textarea',
      'autosize': { minRows: 1, maxRows: 8 },
      'showCount': true,
      'onUpdate:value': (v: string) => {
        editedMessage.value = v
      },
    })
  }

  dialog.warning({
    title: t('common.edit'),
    content: inputNode,
    positiveText: t('common.yes'),
    negativeText: t('common.no'),
    onPositiveClick: () => {
      updateChatSome(+uuid, index, { text: editedMessage.value })
    },
  })
}

function handleClear() {
  if (loading.value)
    return

  dialog.warning({
    title: t('chat.clearChat'),
    content: t('chat.clearChatConfirm'),
    positiveText: t('common.yes'),
    negativeText: t('common.no'),
    onPositiveClick: () => {
      chatStore.clearChatByUuid(+uuid)
    },
  })
}

const showPeople = ref(false)
function handleHIde() {
  showPeople.value = !showPeople.value

  // if (loading.value)
  //   return

  // dialog.warning({
  //   title: t('chat.clearChat'),
  //   content: t('chat.clearChatConfirm'),
  //   positiveText: t('common.yes'),
  //   negativeText: t('common.no'),
  //   onPositiveClick: () => {
  //     chatStore.clearChatByUuid(+uuid)
  //   },
  // })
}


function handleEnter(event: KeyboardEvent) {
  if (!isMobile.value) {
    if (event.key === 'Enter' && !event.shiftKey) {
      event.preventDefault()
      handleSubmit()
    }
  }
  else {
    if (event.key === 'Enter' && event.ctrlKey) {
      event.preventDefault()
      handleSubmit()
    }
  }
}

function handleStop() {
  if (loading.value) {
    homeStore.setMyData({ act: 'abort' })
    controller.abort()
    loading.value = false
  }
}

// 可优化部分
// 搜索选项计算，这里使用value作为索引项，所以当出现重复value时渲染异常(多项同时出现选中效果)
// 理想状态下其实应该是key作为索引项,但官方的renderOption会出现问题，所以就需要value反renderLabel实现
const searchOptions = computed(() => {
  if (prompt.value.startsWith('/')) {
    const abc = promptTemplate.value
      .filter((item: { key: string }) =>
        item.key.toLowerCase().includes(prompt.value.substring(1).toLowerCase()),
      )
      .map((obj: { value: any }) => {
        return {
          label: obj.value,
          value: obj.value,
        }
      })
    mlog('搜索选项', abc)
    return abc
  }
  else if (prompt.value == '@') {
    const abc = gptsUlistStore.myData.slice(0, 10).map((v: gptsType) => {
      return {
        label: v.info,
        gpts: v,
        value: v.gid,
      }
    })
    return abc
  }
  else {
    return []
  }
})


const searchOptions2 = computed(() => {
  if (prompt.value.startsWith('/')) {
    const arr1 = json.mj_commands.map((v: any) => {
      return {
        src: '/midjourney32.webp',
        type: 1,
        label: v.name,
        value: 'mj-' + v.name,
        description: v.description,
      }
    })
    const arr2 = json.niji_commands.map((v: any) => {
      return {
        src: '/nijiourney32.webp',
        type: 2,
        label: v.name,
        value: 'niji-' + v.name,
        description: v.description,
      }
    })

    const all = arr1.concat(arr2)
    const abc = all
      .filter(item =>
        item.label.toLowerCase().includes(prompt.value.substring(1).toLowerCase()),
      )
      .map((obj: any) => {
        return {
          src: obj.src,
          type: obj.type,
          label: obj.label,
          value: obj.value,
          description: obj.description,
        }
      })
    mlog('搜索选项', abc)
    return abc
  }
  return []
});
const goUseGpts = async (item: gptsType) => {
  const saveObj = { model: `${item.gid}`, gpts: item }
  gptConfigStore.setMyData(saveObj)
  if (chatStore.active) {
    // 保存到对话框
    const chatSet = new chatSetting(chatStore.active)
    // if( chatSet.findIndex()>-1 ) chatSet.save( saveObj )
    chatSet.save(saveObj)
  }
  ms.success(t('mjchat.success2'))
  const gptUrl = 'https://gpts.ddaiai.com/open/gptsapi/use'
  myFetch(gptUrl, item)

  mlog('go local ', homeStore.myData.local)
  if (homeStore.myData.local !== 'Chat')
    router.replace({ name: 'Chat', params: { uuid: chatStore.active } })

  gptsUlistStore.setMyData(item)
}

// value反渲染key
const renderOption = (option: { label: string; gpts?: gptsType }) => {
  if (prompt.value == "@") {
    //return [ h( NAvatar,{src:'https://cos.aitutu.cc/gpts/gpt4all.jpg',size:"small",round:true}),option.label ]
    return [
      h(
        "div",
        {
          class: "flex justify-start items-center",
          onclick: () => {
            if (option.gpts) goUseGpts(option.gpts);
            prompt.value = "";
            setTimeout(() => (prompt.value = ""), 80);
          },
        },
        [
          h(NAvatar, {
            src: option.gpts?.logo,
            "fallback-src": "https://cos.aitutu.cc/gpts/3.5net.png",
            size: "small",
            round: true,
            class: "w-8 h-8",
          }),
          h("span", { class: "pl-1" }, option.gpts?.name),
          h(
            "span",
            { class: "line-clamp-1 flex-1 pl-1 opacity-50" },
            option.label
          ),
        ]
      ),
    ];
  }
  for (const i of promptTemplate.value) {
    if (i.value === option.label) return [i.key];
  }
  return [];
};

// value反渲染key
const renderOption2 = (option: any) => {
  // if (prompt.value == '@') {
  //   // return [ h( NAvatar,{src:'https://cos.aitutu.cc/gpts/gpt4all.jpg',size:"small",round:true}),option.label ]
  //   return [
  //     h(
  //       'div',
  //       {
  //         class: 'flex justify-start items-center',
  //         onclick: () => {
  //           if (option.gpts)
  //             goUseGpts(option.gpts)
  //           prompt.value = ''
  //           setTimeout(() => (prompt.value = ''), 80)
  //         },
  //       },
  //       [
  //         h(NAvatar, {
  //           'src': option.gpts?.logo,
  //           'fallback-src': 'https://cos.aitutu.cc/gpts/3.5net.png',
  //           'size': 'small',
  //           'round': true,
  //           'class': 'w-8 h-8',
  //         }),
  //         h('span', { class: 'pl-1' }, option.gpts?.name),
  //         h(
  //           'span',
  //           { class: 'line-clamp-1 flex-1 pl-1 opacity-50' },
  //           option.label,
  //         ),
  //       ],
  //     ),
  //   ]
  // }

  // for (const i of promptTemplate.value) {
  //   if (i.value === option.label)
  //     return [i.key]
  // }


  return [
    h(
      'div',
      {
        class: 'flex justify-start items-center',
        onclick: () => {
          if (option.gpts)
            goUseGpts(option.gpts)
          prompt.value = ''
          setTimeout(() => (prompt.value = ''), 80)
        },
      },
      [
        h(NAvatar, {
          'src': option.src,
          'fallback-src': option.src,
          'size': 'small',
          'round': true,
          'class': 'w-8 h-8',
        }),
        h('span', { class: 'pl-1' }, option.label),
        h(
          'span',
          { class: 'line-clamp-1 flex-1 pl-1 opacity-50' },
          option.description,
        ),
      ],
    ),
  ]
}

const placeholder = computed(() => {
  if (isMobile.value)
    return t('chat.placeholderMobile')
  return t('chat.placeholder')
})

const buttonDisabled = computed(() => {
  return loading.value || !prompt.value || prompt.value.trim() === ''
})

const footerClass = computed(() => {
  let classes = ['p-4']
  if (isMobile.value)
    classes = ['sticky', 'left-0', 'bottom-0', 'right-0', 'p-2', 'pr-3'] // , 'overflow-hidden'
  return classes
})

onMounted(() => {
  scrollToBottom()
  if (inputRef.value && !isMobile.value)
    inputRef.value?.focus()
})

onUnmounted(() => {
  if (loading.value)
    controller.abort()
  homeStore.setMyData({ isLoader: false })
})

const local = computed(() => homeStore.myData.local)
watch(
  () => homeStore.myData.act,
  (n) => {
    if (n == 'draw')
      scrollToBottom()
    if (n == 'scrollToBottom')
      scrollToBottom()
    if (n == 'scrollToBottomIfAtBottom')
      scrollToBottomIfAtBottom()
    if (n == 'gpt.submit' || n == 'gpt.resubmit') {
      loading.value = true
      if (chatStore.active) {
        const chatSet = new chatSetting(chatStore.active)
        if (chatSet.findIndex() == -1) {
          // 如果是空保存到对话框
          chatSet.save(chatSet.getGptConfig())
          setTimeout(() => homeStore.setMyData({ act: 'saveChat' }), 600)
        }
      }
    }
    if (n == 'stopLoading')
      loading.value = false
  },
)
const st = ref({ inputme: true })

watch(
  () => loading.value,
  n => homeStore.setMyData({ isLoader: n }),
)

const ychat = computed(() => {
  let text = prompt.value
  if (loading.value)
    text = ''
  else
    scrollToBottomIfAtBottom()

  return { text, dateTime: t('chat.preview') } as Chat.Chat
})





</script>

<template>
  <div v-if="backgroundImage"
    class=" fixed z-[200] pointer-events-none top-0 left-0 w-full h-full bg-cover bg-center bg-no-repeat" :style="{
      'background-image': `url(${backgroundImage})`,
      'opacity': 0.19,
    }" />
  <div class="flex flex-col w-full h-full">
    <!-- v-if="isMobile" -->
    <HeaderComponent :using-context="usingContext" @export="handleExport" @handle-clear="handleClear"
      @handle-hide="handleHIde" />
    <main class="flex-1 overflow-hidden">
      <div id="scrollRef" ref="scrollRef" class="h-full overflow-hidden overflow-y-auto flex flex-row">
        <div id="image-wrapper" class="w-full max-w-screen-xl m-auto dark:bg-[#101014]"
          :class="[isMobile ? 'p-2' : 'p-4']">
          <template v-if="!dataSources.length">
            <div v-if="
              homeStore.myData.isClient
              && (!gptServerStore.myData.OPENAI_API_BASE_URL
                || !gptServerStore.myData.OPENAI_API_KEY)
            " class="text-center pt-10">
              <AiTextSetting />
            </div>
            <div v-else-if="homeStore.myData.session.notify" class="text-neutral-300 mt-4"
              v-html="homeStore.myData.session.notify" />
            <div v-else class="flex items-center justify-center mt-4 text-center text-neutral-300">
              <SvgIcon icon="ri:bubble-chart-fill" class="mr-2 text-3xl" />
              <span>Aha~</span>
            </div>
          </template>
          <template v-else>
            <div>
              <Message v-for="(item, index) of dataSources" :key="index" :date-time="item.dateTime" :text="item.text"
                :inversion="item.inversion" :error="item.error" :loading="item.loading" :chat="item" :index="index"
                @regenerate="onRegenerate(index)" @delete="handleDelete(index)" @edit="handleEdit(index)" />
              <Message v-if="ychat.text" :key="dataSources.length" :inversion="true" :date-time="$t('mj.typing')"
                :chat="ychat" :text="ychat.text" :index="dataSources.length" />
              <div class="sticky bottom-0 left-0 flex justify-center">
                <NButton v-if="loading" type="warning" @click="handleStop">
                  <template #icon>
                    <SvgIcon icon="ri:stop-circle-line" />
                  </template>
                  {{ t("common.stopResponding") }}
                </NButton>
              </div>
            </div>
          </template>
        </div>

        <div v-if="showPeople" class="h-full sticky top-0">
          <mjSiderPeople :button-disabled="false"></mjSiderPeople>
        </div>
      </div>

    </main>

    <footer v-if="local !== 'draw'" :class="footerClass">
      <div class="w-full max-w-screen-xl m-auto">
        <aiGptInput v-if="
          ['gpt-4-vision-preview', 'gpt-3.5-turbo-16k'].indexOf(
            gptConfigStore.myData.model,
          ) > -1 || st.inputme
        " v-model:modelValue="prompt" :disabled="buttonDisabled" :search-options="searchOptions"
          :render-option="renderOption" />
        <div v-else class="flex items-center justify-between space-x-2">

          <!-- <HoverButton v-if="!isMobile" @click="handleClear">
            <span class="text-xl text-[#4f555e] dark:text-white">
              <SvgIcon icon="ri:delete-bin-line" />
            </span>
          </HoverButton>
          <HoverButton v-if="!isMobile" @click="handleExport">
            <span class="text-xl text-[#4f555e] dark:text-white">
              <SvgIcon icon="ri:download-2-line" />
            </span>
          </HoverButton>
          <HoverButton @click="toggleUsingContext">
            <span class="text-xl" :class="{ 'text-[#4b9e5f]': usingContext, 'text-[#a8071a]': !usingContext }">
              <SvgIcon icon="ri:chat-history-line" />
            </span>
          </HoverButton> -->

          <NAutoComplete v-model:value="prompt" :options="searchOptions" :render-label="renderOption">
            <template #default="{ handleInput, handleBlur, handleFocus }">
              <div class="w-full">
                <NInput ref="inputRef" v-model:value="prompt" type="textarea" :placeholder="placeholder"
                  :autosize="{ minRows: 1, maxRows: isMobile ? 4 : 8 }" @input="handleInput" @focus="handleFocus"
                  @blur="handleBlur" @keypress="handleEnter" />
              </div>
            </template>
          </NAutoComplete>

          <NButton type="primary" :disabled="buttonDisabled" @click="handleSubmit">
            <template #icon>
              <span class="dark:text-black">
                <SvgIcon icon="ri:send-plane-fill" />
              </span>
            </template>
          </NButton>
        </div>
      </div>
    </footer>

    <!-- 绘画 -->
    <footer v-else :class="footerClass">
      <div class="w-full max-w-screen-xl m-auto">
        <!-- <aiGptInput v-if="
          ['gpt-4-vision-preview', 'gpt-3.5-turbo-16k'].indexOf(
            gptConfigStore.myData.model,
          ) > -1 || st.inputme
        " v-model:modelValue="prompt" :disabled="buttonDisabled" :search-options="searchOptions"
          :render-option="renderOption" /> -->
        <div class="flex items-center justify-between space-x-2">



          <NAutoComplete v-model:value="prompt" :options="searchOptions2" :render-label="renderOption2">
            <template #default="{ handleInput, handleBlur, handleFocus }">
              <div class="w-full">
                <NInput size="large" ref="inputRef" v-model:value="prompt" type="textarea" placeholder="给 # 发消息"
                  :autosize="{ minRows: 1, maxRows: isMobile ? 4 : 8 }" @input="handleInput" @focus="handleFocus"
                  @blur="handleBlur" @keypress="handleEnter">
                  <template #prefix>
                    <span class="cursor-pointer hover:text-white mr-2">
                      <svg class="optionIcon_ec5137" aria-hidden="true" role="img" xmlns="http://www.w3.org/2000/svg"
                        width="24" height="24" fill="none" viewBox="0 0 24 24">
                        <path fill="currentColor"
                          d="M13.82 21.7c.17.05.14.3-.04.3H6a4 4 0 0 1-4-4V6a4 4 0 0 1 4-4h7.5c.28 0 .5.22.5.5V5a5 5 0 0 0 5 5h2.5c.28 0 .5.22.5.5v2.3a.4.4 0 0 1-.68.27l-.2-.2a3 3 0 0 0-4.24 0l-4 4a3 3 0 0 0 0 4.25c.3.3.6.46.94.58Z"
                          class=""></path>
                        <path fill="currentColor"
                          d="M21.66 8c.03 0 .05-.03.04-.06a3 3 0 0 0-.58-.82l-4.24-4.24a3 3 0 0 0-.82-.58.04.04 0 0 0-.06.04V5a3 3 0 0 0 3 3h2.66ZM18.3 14.3a1 1 0 0 1 1.4 0l4 4a1 1 0 0 1-1.4 1.4L20 17.42V23a1 1 0 1 1-2 0v-5.59l-2.3 2.3a1 1 0 0 1-1.4-1.42l4-4Z"
                          class=""></path>
                      </svg>
                    </span>
                  </template>
                  <!-- <template #suffix>
                    <NButton type="primary" :disabled="buttonDisabled" @click="handleSubmit">
                      <template #icon>
                        <span class="dark:text-black">
                          <SvgIcon icon="ri:send-plane-fill" />
                        </span>
                      </template>
                    </NButton>
                  </template> -->
                </NInput>
              </div>
            </template>
          </NAutoComplete>

          <NButton style="width: 40px;" size="large" type="primary" :disabled="buttonDisabled" @click="handleSubmit">
            <template #icon>
              <span class="">
                <SvgIcon icon="ri:send-plane-fill" />
              </span>
            </template>
          </NButton>
        </div>
      </div>
    </footer>
  </div>

  <drawListVue />
  <aiGPT @finished="loading = false" />
  <AiSiderInput v-if="isMobile" :button-disabled="false" />
</template>
