<template>
  <div class="layout">
    <t-layout>
      <t-aside key="side" width="78px" :class="`${prefix}-aside`">
          <layout-side-nav :nav-data="sideMenu" />
        </t-aside>
      <t-layout>
        <t-header height="30px" :class="`${prefix}-header`">
          <layout-header />
        </t-header>
        <t-content :class="`${prefix}-content`"><layout-content /></t-content>
      </t-layout>
    </t-layout>
  </div>
</template>

<script setup lang="ts">
import '@/style/layout.less';

import { storeToRefs } from 'pinia';
import { computed } from 'vue';
import { useRoute } from 'vue-router';

import { prefix } from '@/config/global';
import { usePermissionStore } from '@/store';

import LayoutContent from './components/Content.vue';
import LayoutHeader from './components/Header.vue';
import LayoutSideNav from './components/SideNav.vue';

const route = useRoute();

const permissionStore = usePermissionStore();
const { routers: menuRouters } = storeToRefs(permissionStore);

const sideMenu = computed(() => {
  const newMenuRouters = menuRouters.value;
  return newMenuRouters;
});
</script>

<style lang="less" scoped>
.t-layout {
  background: var(--td-bg-color-container);
}
</style>
