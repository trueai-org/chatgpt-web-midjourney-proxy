<script setup lang="ts">
import mjSiderPeopleItem from './mjSiderPeopleItem.vue';
import { useBasicLayout } from '@/hooks/useBasicLayout'
import { NDrawerContent, NDrawer } from "naive-ui";
import { computed, ref } from "vue";
import { homeStore } from '@/store';

//import {  NLayoutSider } from 'naive-ui'; 
//import { SvgIcon } from '@/components/common';

//import { homeStore } from '@/store';
const { isMobile } = useBasicLayout()

const pp = defineProps<{ buttonDisabled: boolean }>()
const st = ref({ show: false })
const handleUpdateCollapsed = (value: boolean) => {
  console.log(value);
}
const $emit = defineEmits(['drawSent', 'close']);
const isLoading = computed(() => {
  return pp.buttonDisabled
})
function drawSent(e: any) {
  st.value.show = false;
  $emit('drawSent', e)
  homeStore.setMyData({ act: 'draw', actData: e });
}

// watch( ()=>homeStore.myData.act, (act) => {
//   act=='newtask' && (st.value.show=true);
//   act=='same2' && (st.value.show=true);
// });

</script>
<template>
  <div v-if="isMobile">
    <n-drawer v-model:show="st.show" :height="565" placement="bottom">
      <n-drawer-content style="--n-body-padding:0" class="h-full">
        <mjSiderPeopleItem @draw-sent="drawSent" :button-disabled="isLoading" />
      </n-drawer-content>
    </n-drawer>
  </div>
  <section class="h-full overflow-auto w-[236px]" @update-collapsed="handleUpdateCollapsed" v-else>
    <div class="h-full w-full">
      <mjSiderPeopleItem @draw-sent="drawSent" :button-disabled="isLoading" />
    </div>
  </section>


</template>