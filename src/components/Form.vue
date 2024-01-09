<template>
  <el-form ref="form" class="form" label-position="top">
    <div style="width: 100%;padding-left: 10px;border-left: 5px solid #2598f8;margin-bottom: 20px;padding-top: 5px;">{{ $t('title') }}</div>
    <el-alert style="margin: 20px 0;background-color: #e1eaff;color: #606266;"  :title="$t('alerts.selectNumberField')" type="info" />

    <!-- 源文本字段  -->
    <el-form-item :label="$t('labels.texts')" size="large" required>
      <div style="width: 100%;" v-if="true">
        <el-select
          
          v-model="toHandleTextsField"
          multiple
          :placeholder="$t('placeholder.texts')"
          style="width: 100%;"
          size="large"
        >
          <el-option
            v-for="item in textFiledList"
            :key="item.id"
            :label="item.name"
            :value="item.id"
          />
        </el-select>
      </div>
    </el-form-item>

    <el-form-item :label="$t('labels.target')" size="large" required>
      <el-select v-model="targetFieldId" :placeholder="$t('placeholder.target')" style="width: 100%">
        <el-option v-for="meta in textFiledList" :key="meta.id" :label="meta.name" :value="meta.id" />
      </el-select>
    </el-form-item>
    
    <el-button style="margin-top: 12px;" v-loading="isWritingData" @click="transfer" :disabled="!issubmitAbled" color="#3370ff" type="primary" plain size="large">{{ $t('submit') }}</el-button>
  </el-form>
</template>

<script setup>
import { FieldType, bitable } from '@lark-base-open/js-sdk';
import { useI18n } from 'vue-i18n';
import { ref, onMounted, computed } from 'vue';


// -- 数据区域
const { t } = useI18n();
const fieldListSeView = ref([]) // 初始化
const textFiledList = ref([]) // 初始化

// 源文本字段选择 - 开始
const toHandleTextsField = ref([])  // 实际用于计算“总交互量”的字段
// 源文本字段选择 - 结束
const targetFieldId = ref('') // 目标

const isWritingData = ref(false) // 提交 - 加载中状态
const issubmitAbled = computed(() => { // 提交 - 禁用状态
  return toHandleTextsField.value.length && targetFieldId.value 
})

// -- 核心算法区域
// --001== 多行数据按次序串联
/** @return {array} */
const spliceTextFieldsContent = async (recordId, toHandleTextsFieldIds) => {
  // 1. for 循环读取每行的数据
  const totalToHandleTextArr = []
  for (let originalTextFieldId of toHandleTextsFieldIds) {
    const originalText = await getCellValueByRFIDS(recordId, originalTextFieldId)
    originalText[originalText.length-1].text += '\n'
    totalToHandleTextArr.push(...originalText)
  }
  const res = totalToHandleTextArr.map(item => item.text)
  console.log("核心算法001 spliceTextFieldsContent() >> return", res)
  return res
}

// --002== 标签包围每行数据
/** @return {array} */
const tagEveryLine = (toHandleTextsList) => {
  // 1. 正则替换 
  const TagedList = toHandleTextsList.map(item => item.replace(/\n$/, '</p>').replace(/^/, '<p>'));
  console.log("tagEveryLine() >> return", TagedList);
  return TagedList
}

// --003== 写入数据
/** @return {object} */
const writeData = async (targetFieldId, recordId, textContent) => {
  const { tableId, viewId } = await bitable.base.getSelection();
  const table = await bitable.base.getActiveTable();

  await table.setCellValue(targetFieldId, recordId, [{ type: 'text', text: textContent }])
  return {"code": 200}
}



// -- 辅助算法区域
// --001== 进行转换，main函数
const transfer = async () => {
  isWritingData.value = true
  // 加载bitable实例
  const { tableId, viewId } = await bitable.base.getSelection();
  const table = await bitable.base.getActiveTable();
  const view = await table.getViewById(viewId);

  // ## mode1: 全部记录
  // const RecordList = await view.getVisibleRecordIdList()

  // ## model2: 交互式选择记录 
  const RecordList = await bitable.ui.selectRecordIdList(tableId, viewId);

  localStorage.setItem('toHandleTextsField', JSON.stringify(toHandleTextsField.value))  // object 类型
  localStorage.setItem('targetFieldId', targetFieldId.value)   // string 类型
  for (let recordId of RecordList) {
    let totalToHandleTextArr = []
    // 拼接
    try {
      totalToHandleTextArr = await spliceTextFieldsContent(recordId, toHandleTextsField.value)
    } catch (error) {
      console.log(error)
      continue
    }
    

    // 标签化
    const tagedArr = tagEveryLine(totalToHandleTextArr)

    // 字符串化
    const stringedTagedArr = tagedArr.map((item, index) => index === tagedArr.length - 1 ? item : item + '\n').join('');

    // 写回数据
    await writeData(targetFieldId.value, recordId, stringedTagedArr)
  }

  isWritingData.value = false
  await bitable.ui.showToast({
    toastType: 'success',
    message: 'success'
  })
  return;
}

// --002== 依据 recordId 和 fieldId 获取 cell Value
/** @param {cellValue} */
const getCellValueByRFIDS = async (recordId, fieldId) => {
  const selection = await bitable.base.getSelection();
  const table = await bitable.base.getTableById(selection.tableId);
  const cellValue = await table.getCellValue(fieldId, recordId)

  return cellValue
}


onMounted(async () => {
  // 获取字段列表 -- start
  const selection = await bitable.base.getSelection()
  const table = await bitable.base.getTableById(selection.tableId)
  const view = await table.getViewById(selection.viewId)
  textFiledList.value = await table.getFieldMetaListByType(FieldType.Text)
  // 获取字段列表 -- end
  
  // 从缓存中获取数据 -- start
  if (localStorage.getItem('toHandleTextsField') !== null) {
    toHandleTextsField.value = JSON.parse(localStorage.getItem('toHandleTextsField')) 
  }
  if (localStorage.getItem('targetFieldId') !== null) {
    targetFieldId.value = localStorage.getItem('targetFieldId')
  }
  // 从缓存中获取数据 -- end
});
    
</script>



<style scoped>
.splice-calss {
  color: #606266;
  font-size: 14px;
}
</style>
