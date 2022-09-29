<template>
  <div>
    <el-form :inline="true" :model="dataForm" :rules="dataRule" ref="dataForm">
      <el-form-item prop="name">
        <el-input
            v-model="dataForm.name"
            placeholder="姓名"
            size="medium"
            class="input"
            clearable="clearable"
        />
      </el-form-item>
      <el-form-item>
        <el-select
            v-model="dataForm.deptId"
            class="input"
            placeholder="部门"
            size="medium"
            clearable="clearable"
        >
          <el-option v-for="one in deptList" :label="one.deptName" :value="one.id"/>
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-select
            v-model="dataForm.typeId"
            class="input"
            placeholder="罚款类型"
            size="medium"
            clearable="clearable"
        >
          <el-option v-for="one in amectTypeList" :label="one.type" :value="one.id"/>
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-date-picker
            v-model="dataForm.date"
            type="daterange"
            range-separator="~"
            start-placeholder="开始日期"
            end-placeholder="结束日期"
            size="medium"
        ></el-date-picker>
      </el-form-item>
      <el-form-item>
        <el-select
            v-model="dataForm.status"
            class="input"
            placeholder="状态"
            size="medium"
            clearable="clearable"
        >
          <el-option label="未缴纳" value="1"/>
          <el-option label="已缴纳" value="2"/>
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button size="medium" type="primary" @click="searchHandle()">查询</el-button>
        <el-button
            size="medium"
            type="primary"
            :disabled="!isAuth(['ROOT', 'AMECT:INSERT'])"
            @click="addHandle()"
        >
          新增
        </el-button>
        <el-button
            size="medium"
            type="danger"
            :disabled="!isAuth(['ROOT', 'AMECT:DELETE'])"
            @click="deleteHandle()"
        >
          批量删除
        </el-button>
        <el-button
            size="medium"
            type="warning"
            :disabled="!isAuth(['ROOT', 'AMECT:SELECT'])"
            @click="reportHandle()"
        >
          查看报告
        </el-button>
      </el-form-item>
    </el-form>
    <el-table
        :data="dataList"
        border
        v-loading="dataListLoading"
        @selection-change="selectionChangeHandle"
        cell-style="padding: 4px 0"
        style="width: 100%;"
        size="medium"
    >
      <el-table-column
          type="selection"
          :selectable="selectable"
          header-align="center"
          align="center"
          width="50"
      />
      <el-table-column width="40px" prop="reason" header-align="center" align="center" type="expand">
        <template #default="scope">
          罚款原因：{{ scope.row.reason }}
        </template>
      </el-table-column>
      <el-table-column type="index" header-align="center" align="center" width="100" label="序号">
        <template #default="scope">
          <span>{{ (pageIndex - 1) * pageSize + scope.$index + 1 }}</span>
        </template>
      </el-table-column>
      <el-table-column prop="type" header-align="center" align="center" label="罚款类型"/>
      <el-table-column prop="name" header-align="center" align="center" label="当事人"/>
      <el-table-column prop="deptName" header-align="center" align="center" label="所属部门"/>
      <el-table-column header-align="center" align="center" label="罚款金额">
        <template #default="scope">
          <span>{{ scope.row.amount }}元</span>
        </template>
      </el-table-column>
      <el-table-column prop="status" header-align="center" align="center" label="状态"/>
      <el-table-column prop="createTime" header-align="center" align="center" label="日期时间"/>
      <el-table-column fixed="right" header-align="center" align="center" width="150" label="操作">
        <template #default="scope">
          <el-button
              type="text"
              size="medium"
              :disabled="!(isAuth(['ROOT', 'AMECT:UPDATE']) && scope.row.status != '已缴纳')"
              @click="updateHandle(scope.row.id)"
          >
            修改
          </el-button>
          <el-button
              type="text"
              size="medium"
              :disabled="!(isAuth(['ROOT', 'AMECT:DELETE']) && scope.row.status != '已缴纳')"
              @click="deleteHandle(scope.row.id)"
          >
            删除
          </el-button>
          <el-button
              type="text"
              size="medium"
              :disabled="!(scope.row.mine == 'true' && scope.row.status == '未缴纳')"
              @click="payHandle(scope.row.id)"
          >
            交款
          </el-button>
        </template>
      </el-table-column>
    </el-table>
    <el-pagination
        @size-change="sizeChangeHandle"
        @current-change="currentChangeHandle"
        :current-page="pageIndex"
        :page-sizes="[10, 20, 50]"
        :page-size="pageSize"
        :total="totalCount"
        layout="total, sizes, prev, pager, next, jumper"
    ></el-pagination>
    <add-or-update v-if="addOrUpdateVisible" ref="addOrUpdate" @refreshDataList="loadDataList"></add-or-update>
    <pay v-if="payVisible" ref="pay" @refreshDataList="loadDataList"></pay>
  </div>
</template>

<script>
import dayjs from 'dayjs';
import AddOrUpdate from './amect-add-or-update.vue';
import Pay from './amect-pay.vue';

export default {
  components: {AddOrUpdate, Pay},
  data: function () {
    return {
      dataForm: {
        name: null,
        deptId: null,
        typeId: null,
        status: null,
        date: null
      },
      deptList: [],
      amectTypeList: [],
      dataList: [],
      pageIndex: 1,
      pageSize: 10,
      totalCount: 0,
      dataListLoading: false,
      dataListSelections: [],
      dataRule: {
        name: [{required: false, pattern: '^[\u4e00-\u9fa5]{1,10}$', message: '姓名格式错误'}]
      },
      addOrUpdateVisible: false,
      payVisible: false
    };
  },
  methods: {
    loadDeptList: function () {
      let that = this;
      that.$http('dept/searchAllDept', 'GET', null, true, function (resp) {
        that.deptList = resp.list;
      });
    },
    loadAmectTypeList: function () {
      let that = this;
      that.$http('amect_type/searchAllAmectType', 'GET', {}, true, function (resp) {
        that.amectTypeList = resp.list;
      });
    },
    //表单数据初始化
    loadDataList: function () {
      this.dataListLoading = true;
      let data = {
        name: this.dataForm.name,
        deptId: this.dataForm.deptId,
        typeId: this.dataForm.typeId,
        status: this.dataForm.status,
        page: this.pageIndex,
        length: this.pageSize
      }
      if (this.dataForm.date != null && this.dataForm.date.length == 2) {
        data.startDate = dayjs(this.dataForm.date[0]).format('YYYY-MM-DD');
        data.endDate = dayjs(this.dataForm.date[1]).format('YYYY-MM-DD');
      }

      this.$http('/amect/searchAmectByPage', 'POST', data, true, res => {
        for (let obj of res.page.list) {
          if (obj.status == 1) {
            obj.status = '未缴纳'
          } else {
            obj.status = '已缴纳'
          }
        }
        this.dataList = res.page.list;
        this.totalCount = res.page.totalCount;
        this.dataListLoading = false;
      })
    },
    //显示条目变更事件
    sizeChangeHandle: function (val) {
      this.pageSize = val;
      this.loadDataList();
    },
    //页码变更事件
    currentChangeHandle: function (val) {
      this.pageIndex = val;
      this.loadDataList();
    },
    //查询事件
    searchHandle: function () {
      this.$refs['dataForm'].validate(valid => {
        if (valid) {
          this.$refs['dataForm'].clearValidate();
          if (this.dataForm.name == '') {
            this.dataForm.name = null;
          }
          this.pageIndex = 1;
          this.loadDataList();
        } else {
          return false;
        }
      })
    },
    //弹出新增窗口
    addHandle: function () {
      this.addOrUpdateVisible = true;
      this.$nextTick(() => {
        this.$refs.addOrUpdate.init();
      })
    },
    //弹出修改窗口
    updateHandle: function (id) {
      this.addOrUpdateVisible = true;
      this.$nextTick(() => {
        this.$refs.addOrUpdate.init(id);
      })
    },
    //复选框选中判定
    selectable: function (row, index) {
      if (row.status != '已缴纳') {
        return true;
      }
      return false;
    },
    //复选框记录保存
    selectionChangeHandle: function (val) {
      this.dataListSelections = val;
    },
    //删除单条罚款记录
    deleteHandle: function (id) {
      this.$confirm('确定删除该条记录吗?', '提示', {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
          }
      ).then(() => {
        let ids = id ? [id] : this.dataListSelections.map(item => {
          return item.id;
        })
        if (ids.length == 0) {
          this.$message({
            message: "没有选中记录",
            type: "warning",
            duration: 1200
          })
        } else {
          this.$http('amect/delete', 'POST', {ids: ids}, true, res => {
            if (res.row > 0) {
              this.$message({
                message: '操作成功',
                type: 'success',
                duration: 1200
              });
              this.loadDataList();
            } else {
              this.$message({
                message: '操作失败',
                type: 'error',
                duration: 1200
              })
            }
          })
        }
      })
    },
    //弹出支付二维码窗口
    payHandle: function (id) {
      this.payVisible = true;
      this.$nextTick(() => {
        this.$refs.pay.init(id);
      });
    }
  },
  created: function () {
    this.loadDeptList();
    this.loadAmectTypeList();
    this.loadDataList();
  }
};
</script>

<style></style>
