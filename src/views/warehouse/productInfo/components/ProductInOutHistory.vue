/*
 * @Author: liucz 
 * @Date: 2018-09-13 10:23:13 
 * @Last Modified by: liucz
 * @Last Modified time: 2018-09-19 17:48:21
 */
<template>
<el-dialog :visible.sync="visibleDialog" @close="" width="75%">
      <el-row class="add-title mb-3">
        <el-col :span="6"><span class="el-dialog__title">历史记录</span></el-col>
        <el-col :span="18" class="text-right">
          <span class="el-dialog__title mr-5">开单日期：{{ formatDate(data.recipt_date, 'yyyy-MM-dd') }}</span>
          <span class="el-dialog__title">单据编号：{{ data.recipt_number }}</span>
        </el-col>
      </el-row>
      <el-row class="toolbar" :span="24">
        <el-col class="text-right">
          <span class="color-999 mr-1">修改时间</span>
          <el-select placeholder="请选择修改时间" size="small" @change="getList"
                    v-model="filters.time" style="width: auto;">
            <el-option v-for="value in timeList" :value="value" :label="value" :key="value"></el-option>
          </el-select>
          <!-- <el-button type="primary" size="medium" @click="getList" icon="el-icon-search">查询</el-button> -->
        </el-col>
      </el-row>
      <el-row class="pt-2">
        <el-col :span=12><span style="font-size:18px">记录人：{{ data.recorder_name }}</span></el-col>
        <el-col :span=12><span style="font-size:18px">入库仓库：{{ data.warehouse_name }}</span></el-col>
      </el-row>
       <el-table :data="list" highlight-current-row
                 v-loading="listLoading"
                 @selection-change="this.sels = sels">
        <el-table-column type="index" :index="getIndex" width="80" label="序号"></el-table-column>
        <el-table-column prop="finish_product_number" label="成品编号" ></el-table-column>
        <el-table-column prop="finish_product_name" label="成品名称" ></el-table-column>
        <el-table-column prop="finish_product_batch_number" label="成品批次号" v-if="module==='in'" ></el-table-column>
        <!-- <el-table-column prop="finish_product_category" label="成品种类" ></el-table-column> -->
        <!-- <el-table-column prop="finish_product_type" label="成品类型" ></el-table-column> -->
        <el-table-column prop="finish_product_model" label="型号" ></el-table-column>
        <!-- <el-table-column prop="finish_product_param" label="成品参数" ></el-table-column> -->
        <el-table-column prop="storage_quantity" label="入库数量" v-if="module==='in'" ></el-table-column>
        <el-table-column prop="finish_delivery_quantity" label="出库数量" v-if="module==='out'"></el-table-column>
        <el-table-column prop="material_unit" label="单位"></el-table-column>
        <el-table-column prop="location_name" label="库位" v-if="module==='in'"></el-table-column>
        </el-table>
         <!--分页-->
        <el-col :span="24" class="toolbar">
          <el-pagination layout="total, sizes, prev, pager ,next" 
                         :page-size="filters.pageSize"
                         :total="filters.total"
                         @current-change="handlePageChange"
                         @size-change="handleSizeChange">
          </el-pagination>
        </el-col>
        <div slot="footer" class="dialog-footer">

        </div>
    </el-dialog>
</template>

<script>
import utils from "@/common/js/utils";
import {
  queryFinishProductStorageHistory,
  queryHistoryTime,
  queryFinishProductDeliveryHistory
} from "@/api/warehouse";
export default {
  name: "ProductInOutHistory",
  data() {
    return {
      filters: {
        time: "",
        reciptNumber: "",
        page: 1,
        pageSize: 10,
        total: 0
      },
      visibleDialog: false,
      listLoading: false,
      timeListLoading: false,
      list: [],
      timeList: []
    };
  },
  props: {
    module: {
      type: String,
      default: "in"
    },
    data: {
      type: Object,
      default: () => {
        return {};
      }
    },
    reciptNumber: {
      type: String,
      required: true
    },
    visible: {
      type: Boolean,
      default: false
    }
  },
  mounted() {},
  watch: {
    visible(val) {
      this.visibleDialog = val;
      setTimeout(() => {
        val && this.getTimeList(this.reciptNumber)
      })
    },
    visibleDialog(val) {
      this.$emit("update:visible", val);
    },
    reciptNumber(val) {
      if (this.visibleDialog) {
        this.getTimeList(val)
      }
    }
  },
  methods: {
    // 修改时间列表
    getTimeList (reciptNumber) {
      this.filters.reciptNumber = reciptNumber
      if (this.timeListLoading) return;
      this.timeListLoading = true
      queryHistoryTime({
        type: this.module === 'in' ? 'CPRK' : 'CPCK',
        reciptNumber: reciptNumber
      }).then(res => {
        if (res.status === 0) {
          res.map.timeList.reverse().forEach((time, index) => {
            res.map.timeList[index] = this.$root.formatDate(time)
          })
          this.timeList = res.map.timeList
          this.timeList = [...new Set(this.timeList)]
          if (this.timeList.length > 0) {
            this.filters.time = this.timeList[0]
          } else {
            this.filters.time = ''
          }
          this.getList();
        } else {
          this.timeList = []
        }
        this.timeListLoading = false
      })
    },
    // 获取列表
    getList() {
      if (this.listLoading) return;
      this.listLoading = true;
      let getListFn = queryFinishProductDeliveryHistory
      if (this.module === 'in') {
        getListFn = queryFinishProductStorageHistory
      }
      getListFn(this.filters).then(res => {
        console.log(res)
        if (res.status === 0) {
          this.filters.total = res.map.line;
          this.list = res.map.finishStorageHistoryDTOs || res.map.finishDeliveryRecordDetailList;
          // console.log(list)
        } else {
          this.list = [];
          this.filters.total = 0;
        }
        this.listLoading = false;
      });
    },
    //  页面改变
    handlePageChange(val) {
      this.filters.page = val;
      this.getList();
    },
    //  每页数量改变
    handleSizeChange(val) {
      this.filters.pageSize = val;
      this.getList();
    },
    // 列表索引
    getIndex(index) {
      return utils.getHeadNumAdapter(
        this.filters.page,
        this.filters.pageSize,
        index
      );
    }
  }
};
</script>

<style scoped>
</style>