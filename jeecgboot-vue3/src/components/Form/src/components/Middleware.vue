<template>
  <div :id="formItemId" class="j-form-item-middleware">
    <slot></slot>
  </div>
</template>

<script setup>
  import { ref } from 'vue';
  // update-begin--author:liaozhiyang---date:20240625---for：【TV360X-1511】blur不生效
  const formItemId = ref(null);
  const props = defineProps(['formName', 'fieldName', 'source']);
  if (props.formName && props.fieldName && props.source !== 'table-query') {
    formItemId.value = `${props.formName}_${props.fieldName}`;
  }
  // update-end--author:liaozhiyang---date:20240625---for：【TV360X-1511】blur不生效
</script>

<style lang="less" scoped>
  // update-begin--author:liaozhiyang---date:20240617---for：【TV360X-1253】代码生成查询区域和新增组件没撑满
  div > :deep(.ant-picker) {
    width: 100%;
  }
  // update-end--author:liaozhiyang---date:20240617---for：【TV360X-1253】代码生成查询区域和新增组件没撑满
</style>
