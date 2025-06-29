<template>
  <!--异步字典下拉搜素-->
  <a-select
    v-if="async"
    v-bind="attrs"
    v-model:value="selectedAsyncValue"
    showSearch
    labelInValue
    allowClear
    :getPopupContainer="getParentContainer"
    :placeholder="placeholder"
    :filterOption="isDictTable ? false : filterOption"
    :notFoundContent="loading ? undefined : null"
    @focus="handleAsyncFocus"
    @search="loadData"
    @change="handleAsyncChange"
    @popupScroll="handlePopupScroll"
    :mode="multiple?'multiple':''"
    @select="handleSelect"
    @deselect="handleDeSelect"
  >
    <template #notFoundContent>
      <a-spin size="small" />
    </template>
    <a-select-option v-for="d in options" :key="d?.value" :value="d?.value">{{ d?.text }}</a-select-option>
  </a-select>
  <!--字典下拉搜素-->
  <a-select
    v-else
    v-model:value="selectedValue"
    v-bind="attrs"
    showSearch
    :getPopupContainer="getParentContainer"
    :placeholder="placeholder"
    :filterOption="filterOption"
    :notFoundContent="loading ? undefined : null"
    :dropdownAlign="{overflow: {adjustY: adjustY }}"
    @change="handleChange"
    :mode="multiple?'multiple':''"
    @select="handleSelect"
    @deselect="handleDeSelect"
  >
    <template #notFoundContent>
      <a-spin v-if="loading" size="small" />
    </template>
    <a-select-option v-for="d in options" :key="d?.value" :value="d?.value">{{ d?.text }}</a-select-option>
  </a-select>
</template>

<script lang="ts">
  import { useDebounceFn } from '@vueuse/core';
  import { defineComponent, PropType, ref, reactive, watchEffect, computed, unref, watch, onMounted } from 'vue';
  import { propTypes } from '/@/utils/propTypes';
  import { useAttrs } from '/@/hooks/core/useAttrs';
  import { initDictOptions } from '/@/utils/dict/index';
  import { defHttp } from '/@/utils/http/axios';
  import { debounce } from 'lodash-es';
  import { setPopContainer } from '/@/utils';
  import { isObject } from '/@/utils/is';

  export default defineComponent({
    name: 'JSearchSelect',
    inheritAttrs: false,
    props: {
      value: propTypes.oneOfType([propTypes.string, propTypes.number]),
      dict: propTypes.string,
      dictOptions: {
        type: Array,
        default: () => [],
      },
      async: propTypes.bool.def(false),
      placeholder: propTypes.string,
      popContainer: propTypes.string,
      pageSize: propTypes.number.def(10),
      getPopupContainer: {
        type: Function,
        default: (node) => node?.parentNode,
      },
      //默认开启Y轴溢出位置调整，因此在可视空间不足时下拉框位置会自动上移，导致Select的输入框被遮挡。需要注意的是，默认情况是是可视空间，而不是所拥有的空间
      //update-begin-author:liusq date:2023-04-04 for:[issue/286]下拉搜索框遮挡问题
      adjustY:propTypes.bool.def(true),
      //update-end-author:liusq date:2023-04-04 for:[issue/286]下拉搜索框遮挡问题
      //是否在有值后立即触发change
      immediateChange: propTypes.bool.def(false),
      //update-begin-author:taoyan date:2022-8-15 for: VUEN-1971 【online 专项测试】关联记录和他表字段 1
      //支持传入查询参数，如排序信息
      params:{
        type: Object,
        default: ()=>{}
      },
      //update-end-author:taoyan date:2022-8-15 for: VUEN-1971 【online 专项测试】关联记录和他表字段 1
      //update-begin---author:wangshuai---date:2025-04-17---for:【issues/8101】前端dict组件导致内存溢出问题：搜索组件支持多选---
      //是否为多选
      multiple:{
        type: Boolean,
        default: false
      },
      //update-end---author:wangshuai---date:2025-04-17---for:【issues/8101】前端dict组件导致内存溢出问题：搜索组件支持多选---
    },
    emits: ['change', 'update:value'],
    setup(props, { emit, refs }) {
      const options = ref<any[]>([]);
      const loading = ref(false);
      // update-begin--author:liaozhiyang---date:20231205---for：【issues/897】JSearchSelect组件添加class/style样式不生效
      const attrs = useAttrs({'excludeDefaultKeys': false});
      // update-end--author:liaozhiyang---date:20231205---for：【issues/897】JSearchSelect组件添加class/style样式不生效
      const selectedValue = ref([]);
      const selectedAsyncValue = ref([]);
      const lastLoad = ref(0);
      // 是否根据value加载text
      const loadSelectText = ref(true);
      // 异步(字典表) - 滚动加载时会用到
      let isHasData = true;
      let scrollLoading = false;
      let pageNo = 1;
      let searchKeyword = '';

      // 是否是字典表
      const isDictTable = computed(() => {
        if (props.dict) {
          return props.dict.split(',').length >= 2
        }
        return false;
      })

      /**
       * 监听字典code
       */
      watch(() => props.dict, () => {
        if (!props.dict) {
          return
        }
        if (isDictTable.value) {
          initDictTableData();
        } else {
          initDictCodeData();
        }
      }, {immediate: true});

      /**
       * 监听value
       */
      watch(
        () => props.value,
        (val) => {
          if (val || val === 0) {
            initSelectValue();
          } else {
            selectedValue.value = [];
            selectedAsyncValue.value = [];
          }
        },
        { immediate: true }
      );
      /**
       * 监听dictOptions
       */
      watch(
        () => props.dictOptions,
        (val) => {
          if (val && val.length >= 0) {
            options.value = [...val];
          }
        },
        { immediate: true }
      );
      /**
       * 异步查询数据
       */
      const loadData = debounce(async function loadData(value) {
        if (!isDictTable.value) {
          return;
        }
        // update-begin--author:liaozhiyang---date:20240731---for：【TV360X-1898】JsearchSelect组件传入字典表格式则支持滚动加载
        pageNo = 1;
        isHasData = true;
        searchKeyword = value;
        // update-end--author:liaozhiyang---date:20240731---for：【TV360X-1898】JsearchSelect组件传入字典表格式则支持滚动加载
 
        lastLoad.value += 1;
        const currentLoad = unref(lastLoad);
        options.value = [];
        loading.value = true;
        let keywordInfo = getKeywordParam(value);
        //update-begin---author:chenrui ---date:2024/4/7  for：[QQYUN-8800]JSearchSelect的search事件在中文输入还没拼字成功时会触发，导致后端SQL注入 #6049------------
        keywordInfo = keywordInfo.replaceAll("'", '');
        //update-end---author:chenrui ---date:2024/4/7  for：[QQYUN-8800]JSearchSelect的search事件在中文输入还没拼字成功时会触发，导致后端SQL注入 #6049------------
        // 字典code格式：table,text,code
        defHttp
          .get({
            url: `/sys/dict/loadDict/${props.dict}`,
            params: { keyword: keywordInfo, pageSize: props.pageSize, pageNo },
          })
          .then((res) => {
            loading.value = false;
            if (res && res.length > 0) {
              if (currentLoad != unref(lastLoad)) {
                return;
              }
              options.value = res;
              // update-begin--author:liaozhiyang---date:20240731---for：【TV360X-1898】JsearchSelect组件传入字典表格式则支持滚动加载
              pageNo++;
              // update-end--author:liaozhiyang---date:20240731---for：【TV360X-1898】JsearchSelect组件传入字典表格式则支持滚动加载
            } else {
              // update-begin--author:liaozhiyang---date:20240731---for：【TV360X-1898】JsearchSelect组件传入字典表格式则支持滚动加载
              pageNo == 1 && (isHasData = false);
              // update-end--author:liaozhiyang---date:20240731---for：【TV360X-1898】JsearchSelect组件传入字典表格式则支持滚动加载
            }
          });
      }, 300);
      /**
       * 初始化value
       */
      function initSelectValue() {
        //update-begin-author:taoyan date:2022-4-24 for: 下拉搜索组件每次选中值会触发value的监听事件，触发此方法，但是实际不需要
        if (loadSelectText.value === false) {
          loadSelectText.value = true;
          return;
        }
        //update-end-author:taoyan date:2022-4-24 for: 下拉搜索组件每次选中值会触发value的监听事件，触发此方法，但是实际不需要
        let { async, value, dict } = props;
        if (async) {
          if (!selectedAsyncValue || !selectedAsyncValue.key || selectedAsyncValue.key !== value) {
            defHttp.get({ url: `/sys/dict/loadDictItem/${dict}`, params: { key: value } }).then((res) => {
              if (res && res.length > 0) {
                //update-begin---author:wangshuai---date:2025-04-17---for:【issues/8101】前端dict组件导致内存溢出问题：搜索组件支持多选---
                //判断组件是否为多选
                if(props.multiple){
                  if(value){
                    let arr: any = [];
                    //多选返回的是以逗号拼接的方式
                    let values = value.toString().split(',');
                    for (let i = 0; i < res.length; i++) {
                      let obj = {
                        key: values[i],
                        label: res[i],
                      };
                      arr.push(obj);
                      selectedValue.value.push(obj.key);
                    }
                    selectedAsyncValue.value = arr;
                  }
                } else {
                  let obj = {
                    key: value,
                    label: res,
                  };
                  if (props.value == value) {
                    selectedAsyncValue.value = { ...obj };
                  }
                  //update-begin-author:taoyan date:2022-8-11 for: 值改变触发change事件--用于online关联记录配置页面
                  if(props.immediateChange == true){
                    emit('change', props.value);
                  }
                  //update-end-author:taoyan date:2022-8-11 for: 值改变触发change事件--用于online关联记录配置页面
                  //update-end---author:wangshuai---date:2025-04-17---for:【issues/8101】前端dict组件导致内存溢出问题：搜索组件支持多选---
                }
              }
            });
          }
        } else {
          //update-begin---author:wangshuai---date:2025-04-17---for:【issues/8101】前端dict组件导致内存溢出问题：搜索组件支持多选---
          if(!props.multiple){
            selectedValue.value = value.toString();
            //update-begin-author:taoyan date:2022-8-11 for: 值改变触发change事件--用于online他表字段配置界面
            if(props.immediateChange == true){
              emit('change', value.toString());
            }
            //update-end-author:taoyan date:2022-8-11 for: 值改变触发change事件--用于online他表字段配置界面
          }else{
            //多选的情况下需要转成数组
            selectedValue.value = value.toString().split(',');
          }
          //update-begin---author:wangshuai---date:2025-04-17---for:【issues/8101】前端dict组件导致内存溢出问题：搜索组件支持多选---
        }
      }

      /**
       * 初始化字典下拉数据
       */
      async function initDictTableData() {
        let { dict, async, dictOptions, pageSize } = props;
        if (!async) {
          //如果字典项集合有数据
          if (dictOptions && dictOptions.length > 0) {
            options.value = dictOptions;
          } else {
            //根据字典Code, 初始化字典数组
            let dictStr = '';
            if (dict) {
              let arr = dict.split(',');
              if (arr[0].indexOf('where') > 0) {
                let tbInfo = arr[0].split('where');
                dictStr = tbInfo[0].trim() + ',' + arr[1] + ',' + arr[2] + ',' + encodeURIComponent(tbInfo[1]);
              } else {
                dictStr = dict;
              }
              //根据字典Code, 初始化字典数组
              const dictData = await initDictOptions(dictStr);
              options.value = dictData;
            }
          }
        } else {
          if (!dict) {
            console.error('搜索组件未配置字典项');
          } else {
            // update-begin--author:liaozhiyang---date:20240731---for：【TV360X-1898】JsearchSelect组件传入字典表格式则支持滚动加载
            pageNo = 1;
            isHasData = true;
            searchKeyword = '';
            // update-end--author:liaozhiyang---date:20240731---for：【TV360X-1898】JsearchSelect组件传入字典表格式则支持滚动加载

            //异步一开始也加载一点数据
            loading.value = true;
            let keywordInfo = getKeywordParam('');
            defHttp
              .get({
                url: `/sys/dict/loadDict/${dict}`,
                params: { pageSize: pageSize, keyword: keywordInfo, pageNo },
              })
              .then((res) => {
                loading.value = false;
                if (res && res.length > 0) {
                  options.value = res;
                  // update-begin--author:liaozhiyang---date:20240731---for：【TV360X-1898】JsearchSelect组件传入字典表格式则支持滚动加载
                  pageNo++;
                  // update-end--author:liaozhiyang---date:20240731---for：【TV360X-1898】JsearchSelect组件传入字典表格式则支持滚动加载
                } else {
                  // update-begin--author:liaozhiyang---date:20240731---for：【TV360X-1898】JsearchSelect组件传入字典表格式则支持滚动加载
                  pageNo == 1 && (isHasData = false);
                  // update-end--author:liaozhiyang---date:20240731---for：【TV360X-1898】JsearchSelect组件传入字典表格式则支持滚动加载
                }
              });
          }
        }
      }

      /**
       * 查询数据字典
       */
      async function initDictCodeData() {
        options.value = await initDictOptions(props.dict);
      }

      /**
       * 同步改变事件
       * */
      function handleChange(value) {
        //update-begin---author:wangshuai---date:2025-04-17---for:【issues/8101】前端dict组件导致内存溢出问题：搜索组件支持多选---
        //多选也会触发change事件，需要判断如果时多选不需要赋值
        if(!props.multiple){
          selectedValue.value = value;
          callback();
        }
        //update-end---author:wangshuai---date:2025-04-17---for:【issues/8101】前端dict组件导致内存溢出问题：搜索组件支持多选---
      }
      /**
       * 异步改变事件
       * */
      function handleAsyncChange(selectedObj) {
          //update-begin---author:wangshuai---date:2025-04-17---for:【issues/8101】前端dict组件导致内存溢出问题：搜索组件支持多选---
          // 单选情况下使用change事件
          if(!props.multiple){
            if (selectedObj) {
              selectedAsyncValue.value = selectedObj;
              selectedValue.value = selectedObj.key;
            } else {
              selectedAsyncValue.value = null;
              selectedValue.value = null;
              options.value = null;
              loadData('');
            }
            callback();
            // update-begin--author:liaozhiyang---date:20240524---for：【TV360X-426】下拉搜索设置了默认值，把查询条件删掉，再点击重置，没附上值
            // 点x清空时需要把loadSelectText设置true
            selectedObj ?? (loadSelectText.value = true);
            // update-end--author:liaozhiyang---date:20240524---for：【TV360X-426】下拉搜索设置了默认值，把查询条件删掉，再点击重置，没附上值
          }
          //update-end---author:wangshuai---date:2025-04-17---for:【issues/8101】前端dict组件导致内存溢出问题：搜索组件支持多选---
      }

      //update-begin---author:wangshuai---date:2025-04-17---for:【issues/8101】前端dict组件导致内存溢出问题：搜索组件支持多选---
      /**
       * 异步值选中事件
       * @param selectedObj
       */
      function handleSelect(selectedObj){
        let key = selectedObj;
        if(props.async){
          key = selectedObj.key;
        }
        //多选情况下使用select事件
        if(props.multiple && key){
          //异步的时候才需要在selectedValue数组中添加值操作，同步的情况下直接走更新值操作
          if(props.async){
            selectedValue.value.push(key);
          }
          selectedObj ?? (loadSelectText.value = true);
          callback();
        }
      }
      
      /**
       * 异步值取消选中事件
       * @param selectedObj
       */
      function handleDeSelect(selectedObj){
        let key = selectedObj;
        if(props.async){
          key = selectedObj.key;
        }
        //多选情况下使用select事件
        if(props.multiple){
          //异步的时候才需要在selectedValue数组中删除值操作，同步的情况下直接走更新值操作
          if(props.async){
            let findIndex = selectedValue.value.findIndex(item => item === key);
            if(findIndex != -1){
              selectedValue.value.splice(findIndex,1);
            }
          }
          selectedObj ?? (loadSelectText.value = true);
          callback();
        }
      }
      //update-end---author:wangshuai---date:2025-04-17---for:【issues/8101】前端dict组件导致内存溢出问题：搜索组件支持多选---
      
      /**
       *回调方法
       * */
      function callback() {
        loadSelectText.value = false;
        //update-begin---author:wangshuai---date:2025-04-17---for:【issues/8101】前端dict组件导致内存溢出问题：搜索组件支持多选---
        //单选直接走更新值操作
        if(!props.multiple){
          emit('change', unref(selectedValue));
          emit('update:value', unref(selectedValue));
        } else {
          //多选需要把数组转成字符串
          emit('change', unref(selectedValue).join(","));
          emit('update:value', unref(selectedValue).join(","));
        }
        //update-end---author:wangshuai---date:2025-04-17---for:【issues/8101】前端dict组件导致内存溢出问题：搜索组件支持多选---
      }
      /**
       * 过滤选中option
       */
      function filterOption(input, option) {
        //update-begin-author:taoyan date:2022-11-8 for: issues/218 所有功能表单的下拉搜索框搜索无效
        let value = '', label = '';
        try {
          value = option.value;
          label = option.children()[0].children;
        }catch (e) {
          console.log('获取下拉项失败', e)
        }
        let str = input.toLowerCase();
        return value.toLowerCase().indexOf(str) >= 0 || label.toLowerCase().indexOf(str) >= 0;
        //update-end-author:taoyan date:2022-11-8 for: issues/218 所有功能表单的下拉搜索框搜索无效
      }

      function getParentContainer(node) {
        // update-begin-author:taoyan date:20220407 for: getPopupContainer一直有值 导致popContainer的逻辑永远走不进去，把它挪到前面判断
        if (props.popContainer) {
          // update-begin--author:liaozhiyang---date:20240517---for：【QQYUN-9339】有多个modal弹窗内都有下拉字典多选和下拉搜索组件时，打开另一个modal时组件的options不展示
          return setPopContainer(node, props.popContainer);
          // update-end--author:liaozhiyang---date:20240517---for：【QQYUN-9339】有多个modal弹窗内都有下拉字典多选和下拉搜索组件时，打开另一个modal时组件的options不展示
        } else {
          if (typeof props.getPopupContainer === 'function') {
            return props.getPopupContainer(node);
          } else {
            return node?.parentNode;
          }
        }
        // update-end-author:taoyan date:20220407 for: getPopupContainer一直有值 导致popContainer的逻辑永远走不进去，把它挪到前面判断
      }

      //update-begin-author:taoyan date:2022-8-15 for: VUEN-1971 【online 专项测试】关联记录和他表字段 1
      //获取关键词参数 支持设置排序信息
      function getKeywordParam(text){
        // 如果设定了排序信息，需要写入排序信息，在关键词后加 [orderby:create_time,desc]
        if(props.params && props.params.column && props.params.order){
          let temp = text||''
          
          //update-begin-author:taoyan date:2023-5-22 for: /issues/4905 表单生成器字段配置时，选择关联字段，在进行高级配置时，无法加载数据库列表，提示 Sgin签名校验错误！ #4905
          temp = temp+'[orderby:'+props.params.column+','+props.params.order+']'
          return encodeURI(temp);
          //update-end-author:taoyan date:2023-5-22 for: /issues/4905 表单生成器字段配置时，选择关联字段，在进行高级配置时，无法加载数据库列表，提示 Sgin签名校验错误！ #4905
          
        }else{
          return text;
        }
      }
      //update-end-author:taoyan date:2022-8-15 for: VUEN-1971 【online 专项测试】关联记录和他表字段 1
      // update-begin--author:liaozhiyang---date:20240523---for：【TV360X-26】下拉搜索控件选中选项后再次点击下拉应该显示初始的下拉选项，而不是只展示选中结果
      const handleAsyncFocus = () => {
        // update-begin--author:liaozhiyang---date:20240709---for：【issues/6681】异步查询不生效
        if ((isObject(selectedAsyncValue.value) || selectedAsyncValue.value?.length) && isDictTable.value && props.async) {
          // update-begin--author:liaozhiyang---date:20240809---for：【TV360X-2062】下拉搜索选择第二页数据后，第一次点击时(得到焦点)滚动条没复原到初始位置且数据会加载第二页数据(应该只加载第一页数据)
          options.value = [];
          // update-end--author:liaozhiyang---date:20240809---for：【TV360X-2062】下拉搜索选择第二页数据后，第一次点击时(得到焦点)滚动条没复原到初始位置且数据会加载第二页数据(应该只加载第一页数据)
          initDictTableData();
        }
        // update-begin--author:liaozhiyang---date:20240919---for：【TV360X-2348】得到焦点时options选项显示第一页内容（解决新增时显示非第一页内容）
        if (Array.isArray(selectedAsyncValue.value) && selectedAsyncValue.value.length === 0 && isDictTable.value && props.async) {
          if (pageNo > 2) {
            options.value = [];
            initDictTableData();
          }
        }
        // update-end--author:liaozhiyang---date:20240919---for：【TV360X-2348】得到焦点时options选项显示第一页内容（解决新增时显示非第一页内容）
        attrs.onFocus?.();
      };
      // update-end--author:liaozhiyang---date:20240523---for：【TV360X-26】下拉搜索控件选中选项后再次点击下拉应该显示初始的下拉选项，而不是只展示选中结果

      /**
       * 2024-07-30
       * liaozhiyang
       * 【TV360X-1898】JsearchSelect组件传入字典表格式则支持滚动加载
       * */
      const handlePopupScroll = async (e) => {
        // 字典表才才支持滚动加载
        if (isDictTable.value) {
          const { target } = e;
          const { scrollTop, scrollHeight, clientHeight } = target;
          if (!scrollLoading && isHasData && scrollTop + clientHeight >= scrollHeight - 10) {
            scrollLoading = true;
            let keywordInfo = getKeywordParam(searchKeyword);

            defHttp
              .get({ url: `/sys/dict/loadDict/${props.dict}`, params: { pageSize: props.pageSize, keyword: keywordInfo, pageNo } })
              .then((res) => {
                loading.value = false;
                if (res?.length > 0) {
                  // 防止开源只更新了前端代码没更新后端代码（第一页和第二页面的第一条数据相同则是后端代码没更新，没分页）
                  if (JSON.stringify(res[0]) === JSON.stringify(options.value[0])) {
                    isHasData =  false;
                    return;
                  }
                  options.value.push(...res);
                  pageNo++;
                } else {
                  isHasData = false;
                }
              })
              .finally(() => {
                scrollLoading = false;
              })
              .catch(() => {
                pageNo != 1 && pageNo--;
              });
          }
        }
      };

      return {
        attrs,
        options,
        loading,
        isDictTable,
        selectedValue,
        selectedAsyncValue,
        loadData: useDebounceFn(loadData, 800),
        getParentContainer,
        filterOption,
        handleChange,
        handleAsyncChange,
        handleAsyncFocus,
        handlePopupScroll,
        handleSelect,
        handleDeSelect,
      };
    },
  });
</script>

<style scoped></style>
