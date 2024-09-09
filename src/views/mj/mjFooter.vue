<script lang="ts" setup>
import { ref } from 'vue'
import { NButton, NForm } from 'naive-ui'
// import { useMjStore } from '@/stores/mj'
// import { MessageContentKey } from '@/content/message'
// import SendIcon from '@/icons/send.vue'

const input = ref('')
const formRef = ref(null)
// const { ins, handleMsg } = useMjStore()
// const ctx = useContext(MessageContentKey)

const handleSubmit = () => {
  // if (!input.value.trim()) {
  //   ctx?.ins.warning('prompt is empty')
  //   return
  // }

  // ctx?.setJobLoading(true)
  // ins.api.imagine(input.value, (type, msg) => handleMsg(type, msg, ctx?.handJobMsg))
  // input.value = ''
}
</script>

<template>
  <div class="fixed bottom-0 inset-x-0 p-2 bg-transparent">
    <NForm ref="formRef" class="flex items-center p-2 rounded-lg bg-gray-600 -top-3 relative"
      @submit.prevent="handleSubmit">
      <NTextarea v-model="input" rows="1" autosize placeholder="send an image prompt"
        class="flex-1 p-1 resize-none bg-transparent outline-none overflow-hidden border-none focus:ring-0" />
      <NButton class="border-l border-gray-500 text-sm h-6 px-2 disabled:text-gray-400" :disabled="!input"
        @click="handleSubmit">
        <SendIcon class="w-4 h-4" />
      </NButton>
    </NForm>
  </div>
</template>

<style scoped>
/* 这里可以添加自定义样式 */
</style>
