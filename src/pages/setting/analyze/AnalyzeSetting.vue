<template>
  <div class="setting-analyze-container">
    <div class="header">
      <t-row justify="space-between">
        <div class="left-operation-container">
          <div class="component-op">
            <div class="item" @click="formDialogVisibleAddAnalyze = true">
              <add-icon />
              <span>添加</span>
            </div>
            <div class="item" @click="removeAllEvent">
              <remove-icon />
              <span>删除</span>
            </div>
            <div class="item" @click="exportEvent">
              <arrow-up-icon />
              <span>导出</span>
            </div>
            <div class="item" @click="flagEvent">
              <discount-icon />
              <span>标识</span>
            </div>
          </div>
        </div>
        <div class="right-operation-container">
          <div class="search">
            <t-input v-model="searchValue" placeholder="搜索解析资源" clearable @enter="getAnalyze" class="search-bar">
              <template #prefix-icon>
                <search-icon size="16px" />
              </template>
            </t-input>
          </div>
        </div>
      </t-row>
    </div>
    <t-table
      row-key="id"
      :data="emptyData ? [] : data"
      :sort="sort"
      height="calc(100vh - 205px)"
      table-layout="auto"
      :columns="COLUMNS"
      :hover="true"
      :pagination="pagination"
      :loading="dataLoading"
      @sort-change="rehandleSortChange"
      @select-change="rehandleSelectChange"
      @page-change="rehandlePageChange"
    >
      <template #name="{ row }">
        <t-badge v-if="row.id === defaultAnalyze" size="small" :offset="[-5, 0]" count="默">{{ row.name }}</t-badge>
        <span v-else>{{ row.name }}</span>
      </template>
      <!-- <template #type="{ row }">
        <span v-if="row.type === 0">普通</span>
        <span v-if="row.type === 1">json</span>
        <span v-if="row.type === 2">多json</span>
        <span v-if="row.type === 3">聚合</span>
      </template> -->
      <template #isActive="{ row }">
        <t-switch v-model="row.isActive" @change="propChangeEvent(row)">
          <template #label="tip">{{ tip.value ? '开' : '关' }}</template>
        </t-switch>
      </template>
      <template #ext="{ row }">
        <span v-for="item in row.ext" :key="item.id">{{ item }},</span>
      </template>
      <template #op="slotProps">
        <a class="t-button-link" @click="defaultEvent(slotProps)">默认</a>
        <a class="t-button-link" @click="editEvent(slotProps)">编辑</a>
        <a class="t-button-link" @click="removeEvent(slotProps)">删除</a>
      </template>
    </t-table>
    <dialog-add-view v-model:visible="formDialogVisibleAddAnalyze" :data="data" @refresh-table-data="getAnalyze" />
    <dialog-edit-view v-model:visible="formDialogVisibleEditAnalyze" :data="formData" />
    <dialog-flag-view
      v-model:visible="formDialogVisibleFlagAnalyze"
      :data="analyzeFlagData"
      @receive-flag-data="setAnalyzeFlag"
    />
  </div>
</template>
<script setup lang="ts">
import { useEventBus } from '@vueuse/core';
import { AddIcon, ArrowUpIcon, DiscountIcon, RemoveIcon, SearchIcon } from 'tdesign-icons-vue-next';
import { MessagePlugin } from 'tdesign-vue-next';
import { onMounted, ref } from 'vue';

import { analyze, setting } from '@/lib/dexie';

import DialogAddView from './components/DialogAdd.vue';
import DialogEditView from './components/DialogEdit.vue';
import DialogFlagView from './components/DialogFlag.vue';
import { COLUMNS } from './constants';

const remote = window.require('@electron/remote');

// Define item form data & dialog status
const formDialogVisibleAddAnalyze = ref(false);
const formDialogVisibleEditAnalyze = ref(false);
const formDialogVisibleFlagAnalyze = ref(false);
const formData = ref();
const defaultAnalyze = ref();
const analyzeFlagData = ref([]);
const sort = ref();
const searchValue = ref();

// Define table
const emptyData = ref(false);
const dataLoading = ref(false);
const data = ref([]);
const pagination = ref({
  defaultPageSize: 20,
  total: 0,
  defaultCurrent: 1,
});
const selectedRowKeys = ref([]);
const rehandleSelectChange = (val) => {
  selectedRowKeys.value = val;
};

onMounted(() => {
  getAnalyze();
  getAnalyzeFlag();
});

const rehandlePageChange = (curr) => {
  pagination.value.defaultCurrent = curr.current;
  pagination.value.defaultPageSize = curr.pageSize;
};

const rehandleSortChange = (sortVal, options) => {
  // sort.value 和 data.value 的赋值都是必须
  sort.value = sortVal;
  data.value = options.currentDataSource;
};

// 获取列表
const getAnalyze = async () => {
  dataLoading.value = true;
  defaultAnalyze.value = await setting.get('defaultAnalyze');
  try {
    const res = await analyze.pagination(searchValue.value);
    if (!res) emptyData.value = true;
    data.value = res.list;
    pagination.value.total = res.total;
  } catch (e) {
    console.log(e);
  } finally {
    dataLoading.value = false;
  }
};

// 获取标识
const getAnalyzeFlag = async () => {
  analyzeFlagData.value = await setting.get('analyzeFlag');
};

// 是否启用
const propChangeEvent = (row) => {
  analyze.update(row.id, { isActive: row.isActive });
};

// 编辑
const editEvent = (row) => {
  formData.value = data.value[row.rowIndex + pagination.value.defaultPageSize * (pagination.value.defaultCurrent - 1)];
  formDialogVisibleEditAnalyze.value = true;
};

const flagEvent = () => {
  formDialogVisibleFlagAnalyze.value = true;
};

const setAnalyzeFlag = (item) => {
  const itemJson = JSON.parse(JSON.stringify(item));
  console.log(itemJson);
  analyzeFlagData.value = itemJson;
  setting.update({ analyzeFlag: itemJson });
};

// 删除
const removeEvent = (row) => {
  analyze
    .remove(row.row.id)
    .then(() => {
      getAnalyze();
      MessagePlugin.success('删除成功');
    })
    .catch((err) => {
      MessagePlugin.error(`删除源失败, 错误信息:${err}`);
    });
};

// 批量删除
const removeAllEvent = () => {
  if (selectedRowKeys.value.length === 0) {
    MessagePlugin.warning('请先选择数据');
    return;
  }
  selectedRowKeys.value.forEach((element) => {
    console.log(element);
    analyze.remove(element).catch((err) => {
      MessagePlugin.error(`批量删除源失败, 错误信息:${err}`);
    });
  });
  getAnalyze();
  MessagePlugin.success('批量删除成功');
};

// 导出接口
const exportEvent = () => {
  const arr = [...data.value];
  const str = JSON.stringify(arr, null, 2);
  const blob = new Blob([str], { type: 'text/plain;charset=utf-8' });
  const reader = new FileReader();
  reader.onload = () => {
    const result: ArrayBuffer = reader.result as ArrayBuffer;
    const buffer = Buffer.from(result);
    remote.dialog
      .showSaveDialog(remote.getCurrentWindow(), {
        defaultPath: 'analyze.json',
        filters: [{ name: 'JSON Files', extensions: ['json'] }],
      })
      .then((saveDialogResult) => {
        if (!saveDialogResult.canceled) {
          const { filePath } = saveDialogResult;
          const fs = remote.require('fs');
          fs.writeFile(filePath, buffer, 'utf-8', (err) => {
            if (err) {
              console.error('Failed to save file:', err);
              MessagePlugin.error('Failed to save file');
            } else {
              console.log('File saved successfully');
              MessagePlugin.success('File saved successfully');
            }
          });
        }
      })
      .catch((err) => {
        console.error('Failed to open save dialog:', err);
        MessagePlugin.error('Failed to open save dialog');
      });
  };
  reader.readAsArrayBuffer(blob);
};

const emitReload = useEventBus<string>('analyze-reload');

// 设置默认接口
const defaultEvent = async (row) => {
  setting.update({
    defaultAnalyze: row.row.id,
  });
  defaultAnalyze.value = row.row.id;
  emitReload.emit('analyze-reload');
  MessagePlugin.success('设置成功');
};
</script>

<style lang="less" scoped>
.setting-analyze-container {
  height: calc(100vh - var(--td-comp-size-l));
  .header {
    margin-bottom: var(--td-comp-margin-s);
  }
  .t-button-link {
    margin-right: var(--td-comp-margin-xxl);
  }
  .left-operation-container {
    .component-op {
      display: flex;
      height: 32px;
      padding: 0 var(--td-comp-paddingLR-xs);
      background-color: var(--td-bg-input);
      backdrop-filter: blur(10px);
      border-radius: var(--td-radius-default);
      align-items: center;
      .item {
        border-radius: var(--td-radius-default);
        transition: all 0.2s ease 0s;
        display: flex;
        align-items: center;
        padding: 2px 4px;
        height: 22px;
        cursor: pointer;
        text-decoration: none;
      }
      .item:hover {
        color: var(--td-text-color-primary);
        background-color: var(--td-bg-color-component-hover);
      }
    }
  }

  :deep(.t-table) {
    background-color: var(--td-bg-color-container);
    tr {
      background-color: var(--td-bg-color-container);
      &:hover {
        background-color: var(--td-bg-color-container-hover);
      }
    }
  }
  :deep(.t-table__header--fixed):not(.t-table__header--multiple) > tr > th {
    background-color: var(--td-bg-color-container);
  }
  :deep(.t-table__pagination) {
    background-color: var(--td-bg-color-container);
  }
}
</style>
