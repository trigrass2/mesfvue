<!--朱缘辉 2018-04-08-->
<template>
  <section>
    <!--搜索 start-->
    <el-col class="toolbar">
      <el-form :inline="true" :model="filters" @submit.native.prevent>
        <el-col :span="4" class="text-left">
          <el-form-item>
            <el-button type="primary" size="medium" @click="handleAdd" icon="el-icon-plus">新增</el-button>
          </el-form-item>
        </el-col>
        <el-col :span="20" class="text-right">
          <el-form-item>
            <el-select placeholder="请选择" size="medium" v-model="filters.projectTypeId" @change="getList">
              <el-option value="" label="全部类型"></el-option>
              <el-option v-for="value in projectTypeLists" :value="value.quality_project_type_id" :label="value.quality_project_type_name" :key="value.quality_project_type_name"></el-option>
            </el-select>
          </el-form-item>
          <el-form-item :span="4">
            <el-input v-model.trim="filters.projectName" size="medium" placeholder="输入关键字" @keyup.native.enter="getList"></el-input>
          </el-form-item>
          <el-form-item>
            <el-button type="primary" size="medium" @click="getList" icon="el-icon-search">查询</el-button>
            <el-button size="medium" v-on:click="reset">重置</el-button>
          </el-form-item>
        </el-col>
      </el-form>
    </el-col>
    <!--搜索 end-->

    <!--列表 start-->
    <el-table :data="projectLists" highlight-current-row
              v-loading="visible.listLoading"
              @selection-change="this.sels = sels">
      <el-table-column type="index" :index="getIndex" width="60"></el-table-column>
      <el-table-column prop="qualityProjects.quality_project_name" label="项目名称" ></el-table-column>
      <el-table-column prop="qualityProjectType.quality_project_type_name" label="项目类型" ></el-table-column>
      <el-table-column label="操作" width="300">
        <template slot-scope="scope">
          <el-button type="primary" size="small" icon="el-icon-edit" @click="handleEdit(scope.row)">修改</el-button>
          <el-button type="danger" size="small" icon="el-icon-delete" @click="handleDelete(scope.row)">删除</el-button>
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

    <!--新增 start-->
    <el-dialog :visible.sync="visible.addForm" :close-on-click-modal="false" width="40%">
      <span slot="title" class="el-dialog__title" v-if="isAddForm">新增项目</span>
      <span slot="title" class="el-dialog__title" v-else>编辑项目</span>
      <el-form :model="addForm" label-width="80px" label-position="right" :rules="addFormRules" ref="addForm" class="clearfix">
        <el-col >
          <el-form-item label="名称" label-width="120px" prop="qualityProjectName">
            <el-input v-model.trim="addForm.qualityProjectName" auto-complete="off" size="medium" clearable></el-input>
          </el-form-item>
        </el-col >
        <el-col >
          <el-form-item label="项目类型" label-width="120px" prop="qualityProjectTypeId">
            <el-select placeholder="请选择" size="medium" v-model="addForm.qualityProjectTypeId" :disabled="!isAddForm">
              <el-option value="" label="全部类型"></el-option>
              <el-option v-for="value in projectTypeLists" :value="value.quality_project_type_id" :label="value.quality_project_type_name" :key="value.quality_project_type_id"></el-option>
            </el-select>
          </el-form-item>
        </el-col >
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button size="medium" @click.native="visible.addForm = false">取消</el-button>
        <el-button type="primary" size="medium" @click.native="addSubmit" :loading="visible.addLoading">提交</el-button>
      </div>
    </el-dialog>
    <!--新增 end-->
  </section>
</template>

<script>
import utils from '../../../common/js/utils'
import {getInspectType, addInspectType, removeInspectType, modifyInspectType, queryProjectTypes} from '../../../api/api'

export default {
  data () {
    return {
      filters: {
        projectName: '',
        projectTypeId: '',
        type: 'info',
        pageSize: 10,
        page: 1,
        total: 0
      },
      projectLists: [],
      projectTypeLists: [],
      options: [
        {value: '1', label: '是'},
        {value: '0', label: '否'}
      ],
      visible: {
        listLoading: false,
        addForm: false,
        addLoading: false,
        editLoading: false
      },
      isAddForm: true,
      addForm: {
        qualityProjectName: '',
        qualityProjectTypeId: '',
        projectId: '',
        isModify: '0'
      },
      modifyForm: {
        projectId: '',
        projectName: ''
      },
      getProjectTypesData: {
        headNum: '',
        projectTypeId: '',
        projectTypeName: ''
      },
      addFormRules: {
        qualityProjectName: { required: true, message: '名称不能为空', trigger: 'blur' },
        qualityProjectTypeId: { required: true, message: '项目类型为必选项', trigger: 'change' }
      }
    }
  },
  computed: {},
  mounted () {
    this.getList()
    this.getProjectTypes()
  },
  methods: {
    //  页面改变
    handlePageChange (val) {
      this.filters.page = val
      this.getList()
    },
    //  每页数量改变
    handleSizeChange (val) {
      this.filters.pageSize = val
      this.getList()
    },
    // 列表索引
    getIndex (index) {
      return parseInt(this.filters.pageSize) * parseInt(this.filters.page - 1) + index + 1
    },
    //  获取列表
    getList () {
      // 注意后端分页用的字段是headNum, 所以要转换一下
      let para = Object.assign({
        headNum: utils.getHeadNumAdapter(this.filters.page, this.filters.pageSize),
      }, this.filters)
      this.visible.listLoading = true
      getInspectType(para).then((res) => {
        if (res.status === 0) {
          // this.filters.page = data.currentPage
          this.filters.total = res.map.lines
          this.projectLists = res.map.projectInfo
        } else {
          this.filters.total = 0
          this.projectLists = []
        }
        this.visible.listLoading = false
      })
    },
    // 获取项目类型列表
    getProjectTypes () {
        queryProjectTypes(this.getProjectTypesData).then((data) => {
          this.projectTypeLists = data.map.projectInfo
        })
    },
    // 新增
    handleAdd () {
      if (this.addForm.qualityProjectTypeId !== '') {
        // 如果用户先点击编辑，再点击新增可能会出现重置到第一次编辑的数据
        this.addForm = {
          qualityProjectName: '',
          qualityProjectTypeId: '',
          isModify: '0'
        }
        this.$refs['addForm'].resetFields()
      }
      this.isAddForm = true
      this.visible.addForm = true
    },
    // 编辑
    handleEdit (row) {
      console.log(row)
      this.addForm = {
        qualityProjectName: row.qualityProjects.quality_project_name,
        qualityProjectTypeId: row.qualityProjectType.quality_project_type_id,
        projectId: row.qualityProjects.quality_project_id,
        isModify: '0'
      }
      this.isAddForm = false
      this.visible.addForm = true
    },
    // 删除
    handleDelete (row) {
      this.$confirm('确定要删除此条数据吗？删除后无法查询').then(action => {
        if (action === 'confirm') {
          removeInspectType({projectId: row.quality_type_project_id}).then((res) => {
            if (res.status === 0) {
              this.$message({ message: '删除成功', type: 'success' })
            } else {
              this.$message({ message: '删除失败', type: 'error' })
            }
            this.visible.addLoading = false
            this.getList()
          })
        }
      }).catch()
    },
    // 重置
    reset () {
      this.filters = {
        projectName: '',
        projectTypeId: '',
        type: 'info',
        pageSize: 10,
        page: 1,
        total: 0
      }
      this.getList()
    },
    // 提交 - 包含新增和编辑
    addSubmit () {
      this.$refs.addForm.validate((valid) => {
        if (valid) {
          this.$confirm('确认提交吗？', '提示', {}).then(() => {
            this.visible.addLoading = true
            let para = Object.assign({}, this.addForm)
            // 给修改传参赋值
            this.modifyForm = {
              projectId: this.addForm.projectId,
              projectName: this.addForm.qualityProjectName
            }
            let para2 = Object.assign({}, this.modifyForm)
            if (this.isAddForm) {
              addInspectType(para).then((res) => {
                this.visible.addLoading = false
                if (res.status === 0) {
                  this.$message({ message: '提交成功', type: 'success' })
                  this.$refs['addForm'].resetFields()
                  this.visible.addForm = false
                  this.getList()
                } else {
                  this.$message({ message: res.msg, type: 'warning' })
                }
              })
            } else {
              modifyInspectType(para2).then((res) => {
                this.visible.addLoading = false
                if (res.status === 0) {
                  this.$message({ message: '提交成功', type: 'success' })
                  this.$refs['addForm'].resetFields()
                  this.visible.addForm = false
                  this.getList()
                } else {
                  this.$message({ message: res.msg, type: 'warning' })
                }
              })
            }
          })
        }
      })
    }
  },
  watch: {}
}
</script>

<style scoped lang="scss">

</style>


