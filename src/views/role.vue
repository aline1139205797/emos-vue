<template>
  <div v-if="isAuth(['ROOT', 'ROLE:SELECT'])">
    <el-form :inline="true" :model="dataForm" :rules="dataRule" ref="dataForm">
      <el-form-item prop="roleName">
        <el-input
            v-model="dataForm.roleName"
            placeholder="角色名称"
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
            :disabled="!isAuth(['ROOT', 'ROLE:INSERT'])"
            @click="addHandle()"
        >
          新增
        </el-button>
        <el-button
            size="medium"
            type="danger"
            :disabled="!isAuth(['ROOT', 'ROLE:DELETE'])"
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
      <el-table-column type="index" header-align="center" align="center" width="100" label="序号">
        <template #default="scope">
          <span>{{ (pageIndex - 1) * pageSize + scope.$index + 1 }}</span>
        </template>
      </el-table-column>
      <el-table-column prop="roleName" header-align="center" align="center" label="角色名称" min-width="180"/>
      <el-table-column header-align="center" align="center" label="权限数量" min-width="140">
        <template #default="scope">
          <span>{{ scope.row.permissions }}个</span>
        </template>
      </el-table-column>
      <el-table-column prop="users" header-align="center" align="center" label="关联用户" min-width="140">
        <template #default="scope">
          <span>{{ scope.row.users }}人</span>
        </template>
      </el-table-column>
      <el-table-column prop="desc" header-align="center" align="center" label="备注" min-width="450"/>
      <el-table-column prop="systemic" header-align="center" align="center" label="内置角色" min-width="100">
        <template #default="scope">
          <span>{{ scope.row.systemic ? '是' : '否' }}</span>
        </template>
      </el-table-column>
      <el-table-column header-align="center" align="center" width="150" label="操作">
        <template #default="scope">
          <el-button
              type="text"
              size="medium"
              :disabled="!isAuth(['ROOT', 'ROLE:UPDATE']) || scope.row.id == 0"
              @click="updateHandle(scope.row.id, scope.row.systemic)"
          >
            修改
          </el-button>
          <el-button
              type="text"
              size="medium"
              :disabled="!isAuth(['ROOT', 'ROLE:DELETE']) || scope.row.systemic || scope.row.users > 0"
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
    <add-or-update v-if="addOrUpdateVisible" ref="addOrUpdate" @refreshDataList="searchHandle"></add-or-update>
  </div>
</template>

<script>
import AddOrUpdate from './role-add-or-update.vue';

export default {
  components: {
    AddOrUpdate
  },
  data: function () {
    return {
      dataForm: {
        roleName: null
      },
      dataList: [],
      pageIndex: 1,
      pageSize: 10,
      totalCount: 0,
      dataListLoading: false,
      dataListSelections: [],
      addOrUpdateVisible: false,
      dataRule: {
        roleName: [{required: false, pattern: '^[a-zA-Z0-9\u4e00-\u9fa5]{1,10}$', message: '角色格式错误'}]
      }
    };
  },
  methods: {
    selectionChangeHandle: function (val) {
      this.dataListSelections = val;
    },
    selectable: function (row, index) {
      if (row.systemic || row.users > 0) {
        return false;
      }
      return true;
    },
    //分页查询角色数据
    searchRoleByPage: function () {
      this.dataListLoading = true;
      let data = {
        "page": this.pageIndex,
        "length": this.pageSize,
        "roleName": this.dataForm.roleName
      }
      this.$http("role/searchRoleByPage", "POST", data, true, res => {
        this.dataList = res.page.list;
        this.totalCount = res.page.totalCount;
        this.dataListLoading = false;
      })
    },
    //条件查询分页数据
    searchHandle: function () {
      this.$refs['dataForm'].validate(valid => {
        if (valid) {
          this.$refs['dataForm'].clearValidate();
          if (this.dataForm.roleName == '') {
            this.dataForm.roleName = null;
          }
          if (this.pageIndex != 1) {
            this.pageIndex = 1;
          }
          this.searchRoleByPage();
        } else {
          return false;
        }
      })
    },
    //条目改变事件
    sizeChangeHandle: function (val){
      this.pageIndex = 1;
      this.pageSize = val;
      this.searchRoleByPage();
    },
    //翻页事件
    currentChangeHandle: function (val){
      this.pageIndex = val;
      this.searchRoleByPage();
    },
    //新增角色弹窗
    addHandle: function () {
      this.addOrUpdateVisible = true;
      this.$nextTick(() => {
        this.$refs.addOrUpdate.init();
      })
    },
    //修改角色弹窗
    updateHandle: function (id, systemic) {
      this.addOrUpdateVisible = true;
      this.$nextTick(() => {
        this.$refs.addOrUpdate.init(id, systemic);
      })
    },
    //删除角色数据
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
          this.$http("role/deleteRolesByIds", "POST", {'ids': ids}, true, res => {
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
    this.searchRoleByPage();
  },
};
</script>

<style></style>
