<!--刘川中 2018.4.9-->
<template>
  <el-row style="background-color: transparent">
    <el-col :span="24">
      <h3>{{ title }}</h3>
    </el-col>
    <!--右边列表-->
    <el-col :span="24">
      <el-col class="toolbar">
        <el-form :inline="true" :model="filters" @submit.native.prevent>
          <el-col :span="4" class="text-left">
            <el-form-item>
              <el-button type="primary" size="medium" @click="handleAdd" icon="el-icon-plus">新增</el-button>
            </el-form-item>
          </el-col>
          <el-col :span="20" class="text-right" hidden>
            <el-form-item :span="4">
              <el-input v-model.trim="filters.productLineName" size="medium" placeholder="输入关键字" @keyup.native.enter="getList"></el-input>
            </el-form-item>
            <el-form-item>
              <el-button type="primary" size="medium" @click="getList" icon="el-icon-search">查询</el-button>
              <el-button size="medium" @click="reset">重置</el-button>
            </el-form-item>
          </el-col>
        </el-form>
      </el-col>
      <!--列表 start-->
      <el-table :data="list" highlight-current-row
                v-loading="visible.listLoading">
        <el-table-column type="index" :index="getIndex" width="60"></el-table-column>
        <el-table-column prop="workstage_name" label="工序名称" sortable></el-table-column>
        <el-table-column prop="workstage_number" label="工序编号" sortable></el-table-column>
        <el-table-column prop="" label="操作">
          <template slot-scope="scope">
            <el-button type="danger" size="small" icon="el-icon-delete" @click="handleDelete(scope.row)">移除</el-button>
          </template>
        </el-table-column>
      </el-table>
      <!--列表 end-->

      <!--分页start -->
      <el-col :span="24" class="toolbar">
        <el-pagination layout="total, sizes, prev, pager ,next" :page-size="filters.pageSize"
                       :total="filters.total"
                       @current-change="handlePageChange" @size-change="handleSizeChange">
        </el-pagination>
      </el-col>
      <!--分页end -->
    </el-col>

  </el-row>
</template>

<script>
  import utils from '../../../common/js/utils'
  import { queryWorkShopInfos, addWorkShopWorkstages, removeWorkShopWorkstages } from '../../../api/api'

  export default {
    name: 'process-config',
    data () {
      return {
        type: '',
        typeId: '',
        title: '',
        filters: {
          // productLineName: '',
          type: 'workstage',
          pageSize: 10,
          page: 1,
          total: 0
        },
        list: [],
        visible: {
        },
        selectedIds: [],
        selectedProcessIds: []
      }
    },
    mounted () {
      this.type = this.$route.query.type
      this.typeId = this.$route.query.id
      this.title = this.$route.query.title
      if (this.type === '' || this.typeId === '') {
        this.$message({ message: '非法访问', type: 'warning' })
        this.$router.back()
      } else {
        this.getList()
        this.getProcessList()
      }
    },
    methods: {
      // 页面改变
      handlePageChange (val) {
        this.filters.page = val
        this.getList()
      },
      // 每页数量改变
      handleSizeChange (val) {
        this.filters.pageSize = val
        this.getList()
      },
      // 列表索引
      getIndex (index) {
        return utils.getHeadNumAdapter(this.filters.page, this.filters.pageSize, index)
      },
      // 重置
      reset () {
        this.filters = {
          // productLineName: '',
          type: 'workstage',
          pageSize: 10,
          page: 1,
          total: 0
        }
        this.getList()
      },
      // 获取列表
      getList () {
        
        let para = Object.assign({
          
        }, this.filters)
        para[this.type] = this.typeId
        this.visible.listLoading = true
        queryWorkShopInfos(para).then((res) => {
          if (res.status === 0) {
            // this.filters.page = data.currentPage
            this.filters.total = res.map.lines
            this.list = res.map.workstages
            this.selectedIds = this.list.map(v => v.workstage_basics_id)
          } else {
            this.filters.total = 0
            this.list = []
            this.selectedIds = []
          }
          this.visible.listLoading = false
        })
      },
      // 获取所有列表
      getProcessList () {
        let para = Object.assign(this.filters, {})
        para[this.type] = this.typeId
        this.visible.listLoading = true
        queryWorkShopInfos(para).then((res) => {
          if (res.status === 0) {
            this.selectedProcessIds = res.map.workstages.map(v => v.workstage_basics_id)
          } else {
            this.selectedProcessIds = []
          }
          this.visible.listLoading = false
        })
      },
      // 新增
      handleAdd () {
        this.$mesProcess.show({
          multiple: true,
          selectedIds: this.selectedProcessIds
        }).then(res => {
          let ids = []
          res.rows.forEach(v => {
            let exist = false
            for (let i = 0, len = this.selectedProcessIds.length; i < len; i++) {
              if (this.selectedProcessIds[i] === v.workstage_basics_id) {
                exist = true
                break
              }
            }
            if (!exist) {
              ids.push(v.workstage_basics_id)
            }
          })
          let para = {
            // type: this.type.split('Id')[0],
            workstageIds: ids.join(',')
          }
          switch (this.type) {
          case 'productLineId':
            para['productId'] = this.typeId
            break;
          case 'workstageId':
            para[this.type] = this.typeId
            break;
          case 'workshopId':
            para[this.type] = this.typeId
            break;
          }
          if (para.workstageIds !== '') {
            addWorkShopWorkstages(para).then(res => {
              if (res.status === 0) {
                this.$message({message: '新增成功', type: 'success'})
                this.getList()
                this.getProcessList()
              } else {
                this.$message({message: res.msg || '新增失败', type: 'error'})
              }
            })
          }
        })
      },
      // 删除
      handleDelete (row) {
        this.$confirm('确定移除吗？').then(action => {
          if (action === 'confirm') {
            let para = {
              workstageIds: row.role_workshop_workstage_id
            }
            // para[this.type] = this.typeId
            removeWorkShopWorkstages(para).then((res) => {
              if (res.status === 0) {
                this.$message({ message: '移除成功', type: 'success' })
              } else {
                this.$message({ message: res.msg || '移除失败', type: 'error' })
              }
              this.visible.addLoading = false
              this.getList()
              this.getProcessList()
            })
          }
        }).catch()
      }
    }
  }
</script>

<style scoped>

</style>
