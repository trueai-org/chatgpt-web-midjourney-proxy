<script setup lang='ts'>
import { computed } from 'vue'
import { NAvatar } from 'naive-ui'
import { useUserStore } from '@/store'
// import defaultAvatar from '@/assets/avatar.jpg'
import { isString } from '@/utils/is'
import { HoverButton, SvgIcon } from '@/components/common'

const userStore = useUserStore()

const userInfo = computed(() => userStore.userInfo)
</script>

<template>
  <div class="flex items-center overflow-hidden content-center">
    <div class="overflow-hidden rounded-full shrink-0 flex items-center mr-3">
      <template v-if="isString(userInfo.avatar) && userInfo.avatar.length > 0">
        <!-- <NAvatar
          size="small"
          round
          :src="userInfo.avatar"
          fallback-src="/discord256.png"
        /> -->
        <SvgIcon icon="ri:openai-fill" class="text-2xl" />
       
      </template>
      <template v-else>
        <SvgIcon icon="ri:openai-fill" class="text-2xl" />
        <!-- <NAvatar size="small" round src="/discord256.png" /> -->
      </template>
    </div>
    <!-- <div class="flex-1 min-w-0 ml-2">
      <h2 class="overflow-hidden text-md text-ellipsis whitespace-nowrap">
        {{ userInfo.name ?? 'AI绘图' }}
      </h2>
      <p class="overflow-hidden text-xs text-gray-500 text-ellipsis whitespace-nowrap">
        <span
          v-if="isString(userInfo.description) && userInfo.description !== ''"
          v-html="userInfo.description"
        />
      </p>
    </div> -->
  </div>
</template>
