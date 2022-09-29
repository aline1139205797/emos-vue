<template>
  <div v-if="isAuth(['ROOT', 'DEPT:SELECT'])">
    <el-form :inline="true" :model="dataForm" :rules="dataRule" ref="dataForm">
      <el-form-item prop="deptName">
        <el-input
            v-model="dataForm.deptName"
            placeholder="部门名称"
            size="medium"
            class="input"
            clearable="clearable"
        />
      </el-form-item>
      <el-form-item>
        <el-button size="medium" type="primary" @click="searchHandle()">查询</el-button>
        <el-button
            size="medium"
            type="primary"
            :disabled="!isAuth(['ROOT', 'DEPT:INSERT'])"
            @click="addHandle()"
        >
          新增
        </el-button>
        <el-button
            size="medium"
            type="danger"
            :disabled="!isAuth(['ROOT', 'DEPT:DELETE'])"
            @click="deleteHandle()"
        >
          批量删除
        </el-button>
      </el-form-item>
    </el-form>
    <el-table
        :data="dataList"
        border
        v-loading="dataListLoading"
        @selection-change="selectionChangeHandle"
        cell-style="padding: 4px 0"
        size="medium"
        style="width: 100%;"
    >
      <el-table-column type="selection" :selectable="selectable" header-align="center" align="center" width="50"/>
      <el-table-column type="index" header-align="center" align="center" width="100" label="序号">
        <template #default="scope">
          <span>{{ (pageIndex - 1) * pageSize + scope.$index + 1 }}</span>
        </template>
      </el-table-column>
      <el-table-column prop="deptName" header-align="center" align="center" label="部门名称" min-width="170"/>
      <el-table-column prop="tel" header-align="center" align="center" label="联系电话" min-width="170"/>
      <el-table-column prop="email" header-align="center" align="center" label="邮箱" min-width="270"/>
      <el-table-column header-align="center" align="center" label="员工数量" min-width="140">
        <template #default="scope">
          <span>{{ scope.row.emps }}人</span>
        </template>
      </el-table-column>
      <el-table-column prop="desc" header-align="center" align="center" label="备注" min-width="400"/>
      <el-table-column header-align="center" align="center" width="150" label="操作">
        <template #default="scope">
          <el-button
              type="text"
              size="medium"
              :disabled="!isAuth(['ROOT', 'DEPT:UPDATE'])"
              @click="updateHandle(scope.row.id)"
          >
            修改
          </el-button>
          <el-button
              type="text"
              size="medium"
              :disabled="!isAuth(['ROOT', 'DEPT:DELETE']) || scope.row.emps > 0"
              @click="deleteHandle(scope.row.id)"
          >
            删除
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
  </div>
</template>

<script>
import AddOrUpdate from './dept-add-or-update.vue';

export default {
  components: {
    AddOrUpdate
  },
  data: function () {
    return {
      dataForm: {
        deptName: null
      },
      dataList: [],
      pageIndex: 1,
      pageSize: 10,
      totalCount: 0,
      dataListLoading: false,
      dataListSelections: [],
      addOrUpdateVisible: false,
      dataRule: {
        deptName: [
          {required: false, pattern: '^[a-zA-Z0-9\u4e00-\u9fa5]{1,10}$', message: '部门名称格式错误'}
        ]
      }
    };
  },
  methods: {
    selectable: function (row, index) {
      if (row.emps > 0) {
        return false
      }
      return true
    },
    selectionChangeHandle: function (val) {
      this.dataListSelections = val;
    },
    //分页数据初始化查询
    loadDataList: function () {
      this.dataListLoading = true;
      let data = {
        'deptName': this.dataForm.deptName,
        'pageIndex': this.pageIndex,
        'pageSize': this.pageSize
      };
      this.$http("dept/searchDeptByPage", "POST", data, true, res => {
        this.dataList = res.page.list;
        this.totalCount = res.page.totalCount;
        this.dataListLoading = false;
      })
    },
    //分页数据查询
    searchHandle: function () {
      this.$refs['dataForm'].validate(valid => {
        if (valid) {
          this.$refs['dataForm'].clearValidate();
          if (this.dataForm.deptName == '') {
            this.dataForm.deptName = null;
          }
          if (this.pageIndex != 1) {
            this.pageIndex = 1;
          }
          this.loadDataList();
        } else {
          return false;
        }
      })
    },
    //页面显示条数变更
    sizeChangeHandle: function (val) {
      this.pageSize = val;
      this.pageIndex = 1;
      this.loadDataList();
    },
    //页面条码变更
    currentChangeHandle: function (val) {
      this.pageIndex = val;
      this.loadDataList();
    },
    //新增部门窗口弹出
    addHandle: function () {
      this.addOrUpdateVisible = true;
      this.$nextTick(() => {
        this.$refs.addOrUpdate.init();
      })
    },
    //修改部门数据
    updateHandle: function (id) {
      this.addOrUpdateVisible = true;
      this.$nextTick(() => {
        this.$refs.addOrUpdate.init(id);
      })
    },
    deleteHandle: function (id) {
      let ids = id ? [id] : this.dataListSelections.map(item => {
        return item.id;
      })
      if (ids.length == 0) {
        this.$message({
          message: '没有选中记录',
          type: 'error',
          duration: 1200
        });
      } else {
        this.$confirm('确定要删除选中的记录?', '提示',
            {
              confirmButtonText: '确定',
              cancelButtonText: '取消',
              type: 'warning',
            }
        ).then(() => {
          this.$http('dept/delete', 'POST', {'ids': ids}, true, res => {
            if (res.rows > 0) {
              this.$message({
                message: '操作成功',
                type: 'success',
                duration: 1200
              });
              this.searchHandle();
            } else {
              this.$message({
                message: '操作失败',
                type: 'error',
                duration: 1200
              });
            }
          })
        })
      }
    }
  },
  created() {
    this.loadDataList();
  }

};
</script>

<style></style>
