<template>
  <el-card>
    <el-tabs v-model="activeName" tab-position="top">
      <el-tab-pane label="基本信息" name="basic">
        <basic-info-form ref="basicInfo" :info="info" />
      </el-tab-pane>
      <el-tab-pane label="生成信息" name="genInfo">
        <gen-info-form ref="genInfo" :info="info" :tables="tables" :columns="columns" />
      </el-tab-pane>
      <el-tab-pane label="字段信息" name="cloum">
        <el-table ref="dragTable" v-loading="loading" :data="columns" row-key="columnId" :max-height="tableHeight">
          <el-table-column label="序号" type="index" min-width="5%" class-name="allowDrag" />
          <el-table-column label="字段列名" prop="columnName" min-width="10%" :show-overflow-tooltip="true" />
          <el-table-column label="字段描述" prop="columnComment" min-width="10%">
            <template #default="scope">
              <el-input v-model="scope.row.columnComment">
              </el-input>
            </template>
          </el-table-column>
          <el-table-column label="物理类型" prop="columnType" min-width="10%" :show-overflow-tooltip="true" />
          <el-table-column label="C#类型" min-width="11%">
            <template #default="scope">
              <el-select v-model="scope.row.csharpType">
                <el-option label="int" value="int" />
                <el-option label="long" value="long" />
                <el-option label="string" value="string" />
                <el-option label="double" value="double" />
                <el-option label="decimal" value="decimal" />
                <el-option label="DateTime" value="DateTime" />
                <el-option label="bool" value="bool" />
              </el-select>
            </template>
          </el-table-column>
          <el-table-column label="C#属性" min-width="10%">
            <template #default="scope">
              <el-input v-model="scope.row.csharpField"></el-input>
            </template>
          </el-table-column>
          <el-table-column label="插入" min-width="5%" v-if="info.tplCategory != 'select'">
            <template #default="scope">
              <el-checkbox v-model="scope.row.isInsert" :disabled="scope.row.isIncrement"></el-checkbox>
            </template>
          </el-table-column>
          <el-table-column label="编辑" min-width="5%" v-if="info.tplCategory != 'select'">
            <template #default="scope">
              <el-checkbox v-model="scope.row.isEdit" :disabled="scope.row.isPk  || scope.row.isIncrement"></el-checkbox>
            </template>
          </el-table-column>
          <el-table-column label="列表" min-width="5%">
            <template #default="scope">
              <el-checkbox v-model="scope.row.isList"></el-checkbox>
            </template>
          </el-table-column>
          <el-table-column label="查询" min-width="5%">
            <template #default="scope">
              <el-checkbox v-model="scope.row.isQuery" :disabled="scope.row.htmlType == 'imageUpload' || scope.row.htmlType == 'fileUpload'">
              </el-checkbox>
            </template>
          </el-table-column>
          <el-table-column label="查询方式" min-width="10%">
            <template #default="scope">
              <el-select v-model="scope.row.queryType" :disabled="scope.row.htmlType == 'datetime'" v-if="scope.row.isQuery">
                <el-option label="=" value="EQ" />
                <el-option label="!=" value="NE" />
                <el-option label=">" value="GT" />
                <el-option label=">=" value="GTE" />
                <el-option label="<" value="LT" />
                <el-option label="<=" value="LTE" />
                <el-option label="LIKE" value="LIKE" />
                <el-option label="BETWEEN" value="BETWEEN" />
              </el-select>
            </template>
          </el-table-column>
          <el-table-column label="必填" min-width="5%">
            <template #default="scope">
              <el-checkbox v-model="scope.row.isRequired"></el-checkbox>
            </template>
          </el-table-column>
          <el-table-column label="表单显示类型" min-width="12%">
            <template #default="scope">
              <el-select v-model="scope.row.htmlType">
                <el-option label="文本框" value="input" />
                <el-option label="数字框" value="inputNumber" />
                <el-option label="文本域" value="textarea" />
                <el-option label="下拉框" value="select" />
                <el-option label="单选框" value="radio" />
                <el-option label="复选框" value="checkbox" />
                <el-option label="日期控件" value="datetime" />
                <el-option label="图片上传" value="imageUpload" />
                <el-option label="文件上传" value="fileUpload" />
                <el-option label="富文本控件" value="editor" />
                <el-option label="自定义输入框" value="customInput" />
              </el-select>
            </template>
          </el-table-column>
          <el-table-column label="字典类型" min-width="12%">
            <template #default="scope">
              <el-select v-model="scope.row.dictType" clearable filterable placeholder="请选择"
                v-if="scope.row.htmlType == 'select' || scope.row.htmlType == 'radio' || scope.row.htmlType =='checkbox'">
                <el-option v-for="dict in dictOptions" :key="dict.dictType" :label="dict.dictName" :value="dict.dictType">
                  <span style="float: left">{{ dict.dictName }}</span>
                  <span style="float: right; color: #8492a6; font-size: 13px">{{ dict.dictType }}</span>
                </el-option>
              </el-select>
            </template>
          </el-table-column>
        </el-table>
      </el-tab-pane>
    </el-tabs>
    <el-form label-width="100px">
      <el-form-item style="text-align: center;margin-left:-100px;margin-top:10px;">
        <el-button type="primary" icon="check" @click="submitForm()">提交</el-button>
        <el-button type="success" icon="refresh" @click="handleQuery()">刷新</el-button>
        <el-button icon="back" @click="close()">返回</el-button>
      </el-form-item>
    </el-form>
  </el-card>
</template>
<script setup name="genedit">
import { updateGenTable, getGenTable } from '@/api/tool/gen'
import { listType } from '@/api/system/dict/type'
import basicInfoForm from './basicInfoForm'
import genInfoForm from './genInfoForm'
import { getCurrentInstance, reactive } from 'vue-demi'
import { useRoute } from 'vue-router'
// import Sortable from 'sortablejs'

// 选中选项卡的 name
const activeName = ref('cloum')
// 表格的高度
const tableHeight = ref(document.documentElement.scrollHeight - 245 + 'px')
// 表信息
const tables = ref([])
// 表列信息
const columns = ref([])
// 字典信息
const dictOptions = ref([])
// 表详细信息
const info = ref({})
const loading = ref(true)

const route = useRoute()
const { proxy } = getCurrentInstance()

function handleQuery() {
  const tableId = route.query && route.query.tableId

  if (tableId) {
    // 获取表详细信息
    getGenTable(tableId).then((res) => {
      loading.value = false
      columns.value = res.data.info.columns
      info.value = res.data.info
      tables.value = res.data.tables // 子表
    })
  }
}
/** 提交按钮 */
function submitForm() {
  const basicForm = proxy.$refs.basicInfo.$refs.basicInfoForm
  const genForm = proxy.$refs.genInfo.$refs.genInfoForm

  Promise.all([basicForm, genForm].map(getFormPromise)).then((res) => {
    const validateResult = res.every((item) => !!item)

    if (validateResult) {
      const genTable = Object.assign({}, info.value)
      genTable.columns = this.columns

      // 额外参数拼接
      genTable.params = {
        treeCode: info.value.treeCode,
        treeName: info.value.treeName,
        treeParentCode: info.value.treeParentCode,
        parentMenuId: info.value.parentMenuId,
        sortField: info.value.sortField,
        sortType: info.value.sortType,
        checkedBtn: info.value.checkedBtn?.toString(),
        permissionPrefix: info.value.permissionPrefix,
      }
      console.log('genForm', genTable)

      updateGenTable(genTable).then((res) => {
        proxy.$modal.msgSuccess(res.msg)
        if (res.code === 200) {
          close()
        }
      })
    } else {
      proxy.$modal.msgError('表单校验未通过，请重新检查提交内容')
    }
  })
}

function getFormPromise(form) {
  return new Promise((resolve) => {
    form.validate((res) => {
      resolve(res)
    })
  })
}

/** 查询字典下拉列表 */
listType({ pageSize: 100 }).then((response) => {
  dictOptions.value = response.data.result
})
/** 关闭按钮 */
function close() {
  const obj = {
    path: '/tool/gen',
    query: { t: Date.now(), pageNum: route.query.pageNum },
  }
  proxy.$tab.closeOpenPage(obj)
}
handleQuery()
</script>