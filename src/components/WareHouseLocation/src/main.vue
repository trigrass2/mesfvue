/*
 * @Author: liucz 
 * @Date: 2018-09-12 10:11:21 
 * @Last Modified by: liucz
 * @Last Modified time: 2018-09-12 14:33:16
 */
<template>
  <el-dialog :title="title || '选择库位' " :visible.sync="visible" class="mes-dialog" width="40%">
    <el-col class="toolbar">
      <el-form :inline="true" :model="params" @submit.native.prevent class="text-right">
        <el-form-item>
          <el-input v-model="params.keyword" placeholder="输入关键字" clearable @clear="getDataList"
            @keyup.native.enter="getDataList" size="small"></el-input>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click.native="getDataList" icon="el-icon-search" size="small">查询</el-button>
        </el-form-item>
      </el-form>
    </el-col>
    <!--列表 start-->
    <el-table :data="list" v-loading="listLoading"
              row-class-name="el-mini-row pointer"
              ref="multipleTable"
              @select="handleRowSelect"
              @select-all="handleSelectAll"
              @selection-change="handleSelectionChange"
              @row-click="toggleCurrentRow"
              :highlight-current-row="!multiple">
      <el-table-column type="selection" width="55" v-if="multiple"></el-table-column>
      <el-table-column type="index" :index="getIndex" label="序号" v-else></el-table-column>
      <el-table-column prop="location_number" label="库位编号" sortable></el-table-column>
      <el-table-column prop="location_name" label="库位名称" sortable></el-table-column>
    </el-table>
    <!--列表 end-->

    <!--分页start -->
    <div class="clearfix hidden">
      <el-col :span="24" class="toolbar">
        <el-pagination layout="total, sizes, prev, pager ,next" :page-size="filters.pageSize"
                       :total="filters.total"
                       @current-change="handlePageChange" @size-change="handleSizeChange">
        </el-pagination>
      </el-col>

      <!--确认选择 start-->
      <el-col :span="24" class="text-right mt-4" v-if="multiple">
        <el-button type="success" size="" @click="confirmSelected" :disabled="!selectedRows || selectedRows.length === 0">确认选择</el-button>
      </el-col>
      <!--确认选择 end-->
    </div>
    <!--分页end -->
  </el-dialog>
</template>

<script>
  import utils from '../../../common/js/utils'
  import { queryLocation } from '../../../api/warehouse'
  export default {
    name: 'warehouse-location',
    data () {
      return {
        params: {
          keyword: '',
          warehouseId: '',
        },
        filters: {
          page: 1,
          pageSize: 99999,
          total: 0
        },
        action: '', // 动作
        visible: false,
        selectedRows: null,
        list: [],
        listLoading: false,
        callback: null,
        title: '',
        multiple: false,
        selectedIds: []
      }
    },
    mounted () {
      // this.getDataList()
    },
    methods: {
      handleSelectionChange (rows) {
        this.selectedRows = rows
        if (this.multiple) {
          this.checkedSelectRows(rows, rows, 'add')
        }
      },
      // 单行选中
      handleRowSelect (rows, row) {
        let checked = false
        rows.map(v => v.location_id).forEach(id => {
          if (id === row.location_id) {
            checked = true
          }
        })
        // 选中 - 新增一行
        if (checked) {
          this.checkedSelectRows(rows, [row], 'add')
        } else {
          this.checkedSelectRows(rows, [row], 'del')
        }
      },
      // 选中全部
      handleSelectAll (rows) {
        // 取消全选
        if (rows.length === 0) {
          this.checkedSelectRows(this.list, this.list, 'del')
        } else {
          this.checkedSelectRows(rows, rows, 'add')
        }
      },
      // 检查行 - 新增还是删除行
      checkedSelectRows (checkedRows, rows, type = 'add') {
        // 新增
        if (type === 'add') {
          this.selectedIds = [...new Set([...this.selectedIds, ...rows.map(v => v.location_id)])]
          // if (this.selectedIds.length > 10) {
          //   this.$message.warning(this.$t(`msg['最多选择10张图片哦']`))
          //   this.selectedIds.splice(10, this.selectedIds.length - 10)
          //   this.toggleSelectionImg(this.list)
          // }
        } else if (type === 'del') {
          // 删除
          rows.forEach(row => {
            let index = this.selectedIds.indexOf(row.location_id)
            if (index > -1) {
              this.selectedIds.splice(index, 1)
            }
          })
        }
      },
      // 确认选择
      confirmSelected (row) {
        this.visible = false
        if (this.multiple) {
          // 选择多行
          this.selectedIds.forEach((id, idx) => {
            this.selectedRows[idx] = { location_id: id }
          })
        } else {
          // 选择单行
          this.selectedRows = [row]
        }
        this.handleAction('selected')
      },
      handleAction (action) {
        this.action = action
        // 手动触发instance的回调，用以resolve
        setTimeout(() => {
          if (this.action) this.callback(this.action, this)
          this.$refs.multipleTable.clearSelection()
        }, 0)
      },
      // 多选情况
      toggleCurrentRow (row) {
        if (this.multiple) {
          this.$refs.multipleTable.toggleRowSelection(row)
          this.$nextTick(() => {
            this.handleRowSelect(this.selectedRows || [], row)
          })
        } else {
          this.confirmSelected(row)
        }
      },
      // 列表索引
      getIndex (index) {
        return utils.getHeadNumAdapter(this.filters.page, this.filters.pageSize, index)
      },
      // 获取数据列表
      getDataList () {
        // this.list = []
        this.listLoading = true
        let para = Object.assign({}, this.filters, this.params)
        queryLocation(para).then((data) => {
          if (data.status === 0) {
            this.filters.total = data.map.line
            this.list = data.map.locations
            this.checkSelectedIds()
          } else {
            this.filters.total = 0
            this.list = []
          }
          this.listLoading = false
        })
      },
      // 初始选择的行
      checkSelectedIds () {
        if (!this.multiple) {
          this.$nextTick(() => {
            this.$refs.multipleTable.setCurrentRow()
          })
        }
        // 多行选择的情况
        if (this.selectedIds.length > 0 && this.multiple) {
          this.selectedIds.forEach(id => {
            this.list.forEach((item, index) => {
              if (item.location_id === id) {
                this.$nextTick(() => {
                  this.$refs.multipleTable.toggleRowSelection(this.list[index], true)
                })
              } else {
                this.$refs.multipleTable.toggleRowSelection(this.list[index], false)
              }
            })
          })
        } else if (this.selectedIds.length > 0 && !this.multiple) {  // 单行选择的情况
          this.list.forEach((item, index) => {
            if (item.location_id === this.selectedIds[0]) {
              this.$nextTick(() => {
                this.$refs.multipleTable.setCurrentRow(this.list[index])
              })
            }
          })
        } else if (this.selectedIds.length === 0) {
          try {
            this.$refs.multipleTable.clearSelection()
          } catch (e) {}
        }
      },
      // 页码改变
      handlePageChange (val) {
        this.filters.page = val
        this.getDataList()
      },
      // 每页数量改变
      handleSizeChange (val) {
        this.filters.pageSize = val
        this.getDataList()
      }
    },
    watch: {
    }
  }
</script>

<style scoped lang="scss">

</style>
