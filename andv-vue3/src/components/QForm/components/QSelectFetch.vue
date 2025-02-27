<template>
  <a-select
    style="width: 100%"
    v-model:value="innerValue"
    showSearch
    v-bind="$attrs"
    :filter-option="filterOption"
    :not-found-content="fetching ? undefined : null"
    @focus="focusHandle()"
    @change="(val, data) => changeHandle(val, data)"
    :getPopupContainer="triggerNode => triggerNode.parentNode"
    @search="searchOptions"
    @popupScroll="e => popupScrollHandle(e, $attrs)"
  >
    <template v-for="item in options" :key="item._value_">
      <a-select-option v-if="item._text_?.toLowerCase().includes(searchValue)" :item="item" :value="item._value_" :text="item._text_">{{ item._text_ }}</a-select-option>
    </template>
    <template v-if="fetching" #notFoundContent>
      <a-spin size="small" />
    </template>
  </a-select>
</template>
<script setup>
import { ref, useAttrs } from 'vue';
const emits = defineEmits(['update:value', 'onChange', 'onFocus']);
const attrs = useAttrs();
const fetching = ref(false);
const options = ref([]);
const searchValue = ref('');
let searchText = '';
const props = defineProps({
  value: {
    type: [String, Number, Array],
    default: () => undefined,
  },
});
const innerValue = computed({
  get: () => props.value,
  set: val => emits('update:value', val),
});
/**
 *
 * _optionsConfig: {
      fetch: QueryAgent,
      optionValue: 'yl0workshopName',
      optionText: 'yl0workshopName',
      paging: 'pageNo',//开启分页
      params: {
      //${key}//会从form中找到key 并替换成value
      //${searchText} 这个是特定的 表示搜索的文本
        className: 'Yl0Workshop',
        queryArgs: {
          attrSet: ['yl0workshopName'],
          condition: [],
          page: {
            pageSize: 10,
            pageNo: 1,
          },
          sort: {
            sortBy: 'createAt',
            sortOrder: 'desc',
          },
        },
      },
    },
 */

function popupScrollHandle(e) {
  const config = attrs._optionsConfig || {};
  if (config.paging) {
    const { target } = e;
    const scrollHeight = target.scrollHeight - target.scrollTop;
    const clientHeight = target.clientHeight;
    const pageConfig = foundHaveKeyObjcet(config.params, config.paging);
    if (scrollHeight - clientHeight < 1) {
      pageConfig[config.paging]++;
      requestOptionsEvent();
    }
  }
}

/**
 * 在嵌套对象中查找包含指定键的对象
 * @param {Object} obj - 要搜索的对象
 * @param {string} key - 要查找的键
 * @param {boolean} iself - 是否仅返回键对应的值，默认为false
 * @returns {Object|null} - 包含指定键的对象或null，如果iself为true，则返回键对应的值
 */
function foundHaveKeyObjcet(obj, key, iself = false) {
  for (const k in obj) {
    if (k === key) {
      if (iself) {
        return obj[k];
      }
      return obj;
    }
    if (Object.prototype.toString.call(obj[k]) === '[object Object]') {
      const result = foundHaveKeyObjcet(obj[k], key);
      if (result) {
        return result;
      }
    }
  }
  return null;
}

initOptionList();

function initOptionList() {
  let event = attrs._optionsConfig?.event || ['mounted'];
  if (event.includes('mounted')) {
    requestOptionsEvent();
  }
}
let timer = null;
function searchOptions(input = '') {
  clearTimeout(timer);
  timer = setTimeout(() => {
    const config = attrs._optionsConfig || {};
    if (config.paging) {
      searchText = input;
      reloadData();
    } else {
      searchValue.value = input;
    }
  }, 300);
}

function reloadData() {
  const config = attrs._optionsConfig || {};
  const pageConfig = foundHaveKeyObjcet(config.params, config.paging);
  if (pageConfig) {
    pageConfig[config.paging] = 1;
  }
  requestOptionsEvent();
}
const filterOption = (input, option) => {
  const config = attrs._optionsConfig || {};
  if (config.paging) {
    return true;
  }
  return option.text.toLowerCase().indexOf(input.toLowerCase()) >= 0;
};

function requestOptionsEvent() {
  const config = attrs._optionsConfig || {};
  if (!config.fetch) {
    return;
  }
  fetching.value = true;

  const params = fetchParamFormat(config.params, { ...attrs.form, searchText });
  config
    .fetch(params || {})
    .then(data => {
      let opData = getValue(data) || [];
      opData = computedSelectOption(opData, config.optionValue || 'objId', config.optionText || 'name');
      if (config.paging) {
        const pageConfig = foundHaveKeyObjcet(config.params, config.paging);
        if (pageConfig[config.paging] > 1) {
          opData = options.value.concat(opData);
        }
      }
      options.value = opData;
    })
    .finally(() => {
      fetching.value = false;
    });
}
function extractTemplates(str) {
  const regex = /\${([^}]+)}/g; // 匹配大括号内的内容
  let match;
  const templates = [];

  while ((match = regex.exec(str))) {
    templates.push(match[1]); // 添加到结果数组中，不包括大括号
  }
  return templates;
}
function fetchParamFormat(params = {}, form = {}) {
  let paramStr = JSON.stringify(params);
  const templates = extractTemplates(paramStr) || [];
  paramStr = templates.reduce((string, item) => {
    return globalReplace(string, `\${${item}}`, form[item] || '');
  }, paramStr);
  return JSON.parse(paramStr);
}

function globalReplace(string, search, replacement) {
  // 创建正则表达式，并添加全局标志（g）
  const regex = new RegExp('\\' + search, 'g');
  // 执行替换
  return string.replace(regex, replacement);
}

function filterFactories(template = [], searchForm = {}) {
  for (let i = 0; i < template.length; i++) {
    let item = template[i];
    let value = searchForm[item.proxyField] || searchForm[item.key];
    if (value && !item.isearch) {
      item.value = value;
      delete item.proxyField;
    } else if (!item.value) {
      template.splice(i, 1);
      i--;
    }
  }
  console.log(template);
}

function searchParams(filterTemplate = {}, searchForm = {}) {
  const template = JSON.parse(JSON.stringify(filterTemplate));
  filterFactories(template, searchForm);
  return template;
}

const getValue = data => {
  for (const key of Object.keys(data)) {
    if (Array.isArray(data[key])) {
      return data[key];
    }
    if (Object.prototype.toString.call(data[key]) === '[object Object]') {
      const result = getValue(data[key]);
      if (result) {
        return result;
      }
    }
  }
};
function computedSelectOption(data, value = 'id', text = 'name') {
  const options = [];
  if (Array.isArray(data)) {
    for (let i = 0; i < data.length; i++) {
      let val = '';
      let txt = '';
      if (value.includes('{{')) {
        val = render(value, data[i]);
      }
      if (text.includes('{{')) {
        txt = render(text, data[i]);
      }
      const item = {
        _value_: val || data[i][value],
        _text_: txt || data[i][text],
      };
      options.push(item);
    }
  }
  return options;
}

function render(template, person) {
  let reg = /{{(.*?)}}/g;
  let res = template.replace(reg, (item, key) => {
    return person[key];
  });
  return res;
}

function focusHandle() {
  let event = attrs._optionsConfig?.event || [];
  emits('onFocus', innerValue.value);
  if (event.includes('focus') && !innerValue.value) {
    reloadData();
  }
}
function changeHandle(val, data) {
  emits('onChange', val, data);
  if (attrs.hasOwnProperty('_beLink')) {
    attrs.form[attrs._beLink] = undefined;
  }
  if (attrs.hasOwnProperty('_padField')) {
    let obj = data.item;
    for (let i in attrs._padField) {
      attrs.form[i] = obj[attrs._padField[i]];
    }
  }
}
</script>
