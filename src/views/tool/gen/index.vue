<template>
  <div class="app-container">

    <el-form ref="codeform" :inline="true" :model="queryParams">
      <el-form-item label="表名" prop="tableName">
        <el-input v-model="queryParams.tableName" clearable placeholder="输入要查询的表名" />
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="search" @click="getList()">查询</el-button>
        <!-- <el-button type="default" icon="refresh" @click="loadTableData()">刷新</el-button> -->
      </el-form-item>
    </el-form>

    <el-row :gutter="10" class="mb10">
      <el-col :span="1.5">
        <el-button type="info" plain icon="upload" @click="openImportTable" v-hasPermi="['tool:gen:import']">导入</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button type="danger" :disabled="multiple" plain icon="delete" @click="handleDelete" v-hasPermi="['tool:gen:remove']">
          删除</el-button>
      </el-col>
    </el-row>
    <el-table ref="gridtable" v-loading="tableloading" :data="tableList" border @selection-change="handleSelectionChange" highlight-current-row
      height="480px">
      <el-table-column type="selection" align="center" width="55"></el-table-column>
      <el-table-column label="序号" type="index" width="50" align="center">
        <template #default="scope">
          <span>{{(queryParams.pageNum - 1) * queryParams.pageSize + scope.$index + 1}}</span>
        </template>
      </el-table-column>
      <el-table-column prop="dbName" label="数据库名" width="90" />
      <el-table-column prop="tableId" label="表id" width="70" sortable="" />
      <el-table-column prop="tableName" label="表名" width="110" :show-overflow-tooltip="true" />
      <el-table-column prop="tableComment" label="表描述" :show-overflow-tooltip="true" width="120" />
      <el-table-column prop="className" label="实体" :show-overflow-tooltip="true" />
      <el-table-column prop="createTime" label="创建时间" />
      <el-table-column prop="updateTime" label="更新时间" />
      <el-table-column label="操作" align="center" width="320">
        <template #default="scope">
          <el-button type="text" icon="view" @click="handlePreview(scope.row)" v-hasPermi="['tool:gen:preview']">预览</el-button>
          <el-button type="text" icon="edit" @click="handleEditTable(scope.row)" v-hasPermi="['tool:gen:edit']">编辑</el-button>
          <el-button type="text" icon="delete" @click="handleDelete(scope.row)" v-hasPermi="['tool:gen:remove']">删除</el-button>
          <el-button type="text" icon="refresh" @click="handleSynchDb(scope.row)" v-hasPermi="['tool:gen:edit']">同步</el-button>
          <el-button type="text" icon="download" @click="handleGenTable(scope.row)" v-hasPermi="['tool:gen:code']">生成代码
          </el-button>
        </template>
      </el-table-column>
    </el-table>
    <pagination :page="queryParams.pageNum" v-model:limit="queryParams.pageSize" v-model:total="total" @pagination="getList" />

    <!-- 预览界面 -->
    <el-dialog :title="preview.title" v-model="preview.open" width="80%" top="5vh" append-to-body>
      <el-tabs v-model="preview.activeName">
        <el-tab-pane v-for="(item, key) in preview.data" :label="item.title" :name="key.toString()" :key="key">
          <el-link :underline="false" icon="document-copy" v-clipboard:copy="item.content" v-clipboard:success="clipboardSuccess" style="float:right">
            复制</el-link>
          <pre><code class="hljs" v-html="highlightedCode(item.content, item.title)"></code></pre>
        </el-tab-pane>
      </el-tabs>
    </el-dialog>
    <import-table ref="importRef" @ok="getList" />
  </div>
</template>

<script setup name="gen">
import {
  codeGenerator,
  listTable,
  delTable,
  previewTable,
  synchDb,
} from '@/api/tool/gen'
import { useRouter } from 'vue-router'
import importTable from './importTable'
import hljs from 'highlight.js'
import 'highlight.js/styles/idea.css' // 这里有多个样式，自己可以根据需要切换

// const route = useRoute()
const router = useRouter()
const { proxy } = getCurrentInstance()

const tableList = ref([])
const tableloading = ref(true)
const showSearch = ref(true)
const tableIds = ref([])
const single = ref(true)
const multiple = ref(true)
const total = ref(0)
const tableNames = ref([])
const dateRange = ref([])
const uniqueId = ref('')
const showGenerate = ref(false)
// 选中行的表
const currentSelected = ref({})

const data = reactive({
  queryParams: {
    pageNum: 1,
    pageSize: 10,
    tableName: undefined,
    // tableComment: undefined,
  },
  preview: {
    open: false,
    title: '代码预览',
    data: {},
    activeName: '0',
  },
})

const { queryParams, preview } = toRefs(data)

// onActivated(() => {
//   const time = route.query.t
//   if (time != null && time != uniqueId.value) {
//     uniqueId.value = time
//     queryParams.value.pageNum = Number(route.query.pageNum)
//     dateRange.value = []
//     proxy.resetForm('queryForm')
//     getList()
//   }
// })

/** 查询表集合 */
function getList() {
  tableloading.value = true
  listTable(proxy.addDateRange(queryParams.value, dateRange.value)).then(
    (response) => {
      tableList.value = response.data.result
      total.value = response.data.totalNum
      tableloading.value = false
    }
  )
}
/** 搜索按钮操作 */
function handleQuery() {
  queryParams.value.pageNum = 1
  getList()
}
/** 生成代码操作 */
function handleGenTable(row) {
  currentSelected.value = row
  if (!currentSelected.value) {
    proxy.$modal.msgError('请先选择要生成代码的数据表')
    return false
  }
  proxy.$refs['codeform'].validate((valid) => {
    if (valid) {
      proxy.$modal.loading('正在生成代码...')

      var seachdata = {
        tableId: currentSelected.value.tableId,
        tableName: currentSelected.value.name,
      }

      codeGenerator(seachdata)
        .then((res) => {
          const { data } = res
          showGenerate.value = false
          if (row.genType === '1') {
            proxy.$modal.msgSuccess('成功生成到自定义路径')
          } else {
            proxy.$modal.msgSuccess('恭喜你，代码生成完成！')
            proxy.download(data.path)
          }
          proxy.$modal.closeLoading()
        })
        .catch((erre) => {
          proxy.$modal.closeLoading()
        })
    } else {
      return false
    }
  })
}
/** 同步数据库操作 */
function handleSynchDb(row) {
  const tableName = row.tableName
  proxy
    .$confirm('确认要强制同步"' + tableName + '"表结构吗？')
    .then(function () {
      return synchDb(row.tableId, { tableName, dbName: row.dbName })
    })
    .then(() => {
      proxy.$modal.msgSuccess('同步成功')
    })
    .catch(() => {})
}
/** 打开导入表弹窗 */
function openImportTable() {
  proxy.$refs['importRef'].show()
}
/** 重置按钮操作 */
function resetQuery() {
  dateRange.value = []
  proxy.resetForm('queryParams')
  handleQuery()
}
/** 预览按钮 */
function handlePreview(row) {
  proxy.$refs['codeform'].validate((valid) => {
    if (!valid) {
      proxy.$modal.msgError('请先完成表格')
      return
    }
    proxy.$modal.loading('请稍后...')
    previewTable(row.tableId).then((res) => {
      if (res.code === 200) {
        showGenerate.value = false
        preview.value.open = true
        preview.value.data = res.data
        proxy.$modal.closeLoading()
      }
    })
  })
}
// 多选框选中数据
function handleSelectionChange(selection) {
  tableIds.value = selection.map((item) => item.tableId)
  // tableNames.value = selection.map((item) => item.tableName)
  // single.value = selection.length != 1
  multiple.value = !selection.length
}
/** 编辑表格 */
function handleEditTable(row) {
  queryParams.value.tableName = row.tableName
  getList()

  router.push({
    path: '/gen/editTable',
    query: { tableId: row.tableId },
  })
}
/** 删除按钮操作 */
function handleDelete(row) {
  const tableIds = row.tableId || tableIds.value
  proxy
    .$confirm('此操作将永久删除该文件, 是否继续?', '提示', {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'warning',
    })
    .then(() => {
      delTable(tableIds.toString()).then((res) => {
        if (res.code == 200) {
          proxy.$modal.msgSuccess('删除成功')

          handleQuery()
        }
      })
    })
    .catch(() => {
      proxy.$message({
        type: 'info',
        message: '已取消删除',
      })
    })
}
/** 高亮显示 */
function highlightedCode(code, key) {
  // var language = key.substring(key.lastIndexOf(".") , key.length)
  const result = hljs.highlightAuto(code || '')
  return result.value || '&nbsp;'
}
/** 复制代码成功 */
function clipboardSuccess() {
  proxy.$modal.msgSuccess('复制成功')
}
function cancel() {
  showGenerate.value = false
  currentSelected.value = {}
}
getList()
</script>