<script setup lang="ts">
import { NTabs, NTabPane, NTag, NAvatar } from 'naive-ui';
import aiDrawInputItem from './aiDrawInputItem.vue'
import aiFace from './aiFace.vue'
import aiBlend from './aiBlend.vue'
import aiDall from './aiDall.vue'
import { useBasicLayout } from '@/hooks/useBasicLayout'
import { SvgIcon } from '@/components/common'
const $emit = defineEmits(['drawSent', 'close']);
const drawSent = (d: any) => $emit('drawSent', d);
const { isMobile } = useBasicLayout()
</script>

<template>
  <div class="overflow-y-auto bg-[#fafbfc] pt-2 dark:bg-[#18181c] h-full ">

    <div class="flex flex-col px-2 pt-3 text-gray-400">

      <div
        class="flex flex-row items-center space-x-2 cursor-pointer hover:bg-gray-100 dark:hover:bg-black/50 hover:rounded p-2">
        <NAvatar size="small" src="/midjourney32.webp" round />
        <div class="truncate">
          Midjourney Bot
        </div>
        <NTag size="small" type="info">
          <template #icon>
            <SvgIcon icon="ri:check-line" />
          </template>
          APP
        </NTag>
      </div>
      <div
        class="flex flex-row items-center space-x-2 cursor-pointer hover:bg-gray-100 dark:hover:bg-black/50 hover:rounded p-2">
        <NAvatar size="small" src="/nijiourney32.webp" round />
        <div class="truncate">
          niji・journey Bot
        </div>
        <NTag size="small" type="info">
          <template #icon>
            <SvgIcon icon="ri:check-line" />
          </template>
          APP
        </NTag>
      </div>
    </div>
  </div>
</template>