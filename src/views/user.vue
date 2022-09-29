<template>
  <div v-if="isAuth(['ROOT', 'USER:SELECT'])">
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
        <el-select v-model="dataForm.sex" class="input" placeholder="性别" size="medium" clearable="clearable">
          <el-option label="男" value="男"/>
          <el-option label="女" value="女"/>
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-select v-model="dataForm.role" class="input" placeholder="角色" size="medium" clearable="clearable">
          <el-option v-for="one in roleList" :label="one.roleName" :value="one.roleName"/>
        </el-select>
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
            v-model="dataForm.status"
            class="input"
            placeholder="状态"
            size="medium"
            clearable="clearable"
        >
          <el-option label="在职" value="1"/>
          <el-option label="离职" value="2"/>
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button size="medium" type="primary" @click="searchHandle()">查询</el-button>
        <el-button
            size="medium"
            type="primary"
            :disabled="!isAuth(['ROOT', 'USER:INSERT'])"
            @click="addHandle()"
        >新增
        </el-button>
        <el-button
            size="medium"
            type="danger"
            :disabled="!isAuth(['ROOT', 'USER:DELETE'])"
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
      <el-table-column type="selection" header-align="center" align="center" width="50"/>
      <el-table-column type="index" header-align="center" align="center" width="100" label="序号">
        <template #default="scope">
          <span>{{ (pageIndex - 1) * pageSize + scope.$index + 1 }}</span>
        </template>
      </el-table-column>
      <el-table-column prop="name" header-align="center" align="center" min-width="100" label="姓名"/>
      <el-table-column prop="sex" header-align="center" align="center" min-width="60" label="性别"/>
      <el-table-column prop="tel" header-align="center" align="center" min-width="130" label="电话"/>
      <el-table-column
          prop="email"
          header-align="center"
          align="center"
          min-width="240"
          label="邮箱"
          :show-overflow-tooltip="true"
      />
      <el-table-column prop="hiredate" header-align="center" align="center" min-width="130" label="入职日期"/>
      <el-table-column
          prop="roles"
          header-align="center"
          align="center"
          min-width="150"
          label="角色"
          :show-overflow-tooltip="true"
      />
      <el-table-column prop="deptName" header-align="center" align="center" min-width="120" label="部门"/>
      <el-table-column prop="status" header-align="center" align="center" min-width="100" label="状态"/>
      <el-table-column header-align="center" align="center" width="150" label="操作">
        <template #default="scope">
          <el-button
              type="text"
              size="medium"
              v-if="isAuth(['ROOT', 'USER:UPDATE'])"
              @click="updateHandle(scope.row.id)"
          >
            修改
          </el-button>
          <el-button
              type="text"
              size="medium"
              v-if="isAuth(['ROOT', 'USER:UPDATE'])"
              :disabled="scope.row.status == '离职' || scope.row.root"
              @click="dimissHandle(scope.row.id)"
          >
            离职
          </el-button>
          <el-button
              type="text"
              size="medium"
              :disabled="scope.row.root"
              v-if="isAuth(['ROOT', 'USER:DELETE'])"
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
    <dimiss v-if="dimissVisible" ref="dimiss" @refreshDataList="loadDataList"></dimiss>
  </div>
</template>

<script>
import AddOrUpdate from './user-add-or-update.vue';
import Dimiss from './dimiss.vue';

export default {
  components: {
    AddOrUpdate,
    Dimiss
  },
  data() {
    return {
      dataForm: {
        name: '',
        sex: '',
        role: '',
        deptId: '',
        status: ''
      },
      dataList: [],
      roleList: [],
      deptList: [],
      pageIndex: 1,
      pageSize: 10,
      totalCount: 0,
      dataListLoading: false,
      dataListSelections: [],
      addOrUpdateVisible: false,
      dimissVisible: false,
      dataRule: {
        name: [{required: false, pattern: '^[\u4e00-\u9fa5]{1,10}$', message: '姓名格式错误'}]
      }
    };
  },
  methods: {
    loadRoleList: function () {
      let that = this;
      that.$http('role/searchAllRole', 'GET', null, true, function (resp) {
        that.roleList = resp.list;
      });
    },
    loadDeptList: function () {
      let that = this;
      that.$http('dept/searchAllDept', 'GET', null, true, function (resp) {
        that.deptList = resp.list;
      });
    },
    selectionChangeHandle: function (val) {
      this.dataListSelections = val;
    },
    //分页数据查询
    searchUserByPage: function () {
      this.dataListLoading = true;
      let data = {
        'pageIndex': this.pageIndex,
        'pageSize': this.pageSize,
        'name': this.dataForm.name,
        'sex': this.dataForm.sex,
        'role': this.dataForm.role,
        'deptId': this.dataForm.deptId,
        'status': this.dataForm.status
      }
      this.$http('user/getUserPage', 'POST', data, true, res => {
        for (let dataObj of res.page.list) {
          if (dataObj.status == 1) {
            dataObj.status = "在职";
          } else if (dataObj.status == 2) {
            dataObj.status = "离职";
          }
        }
        this.dataList = res.page.list;
        this.totalCount = res.page.totalCount;
        this.pageSize = res.page.pageSize;
        this.pageIndex = res.page.pageIndex;
        this.dataListLoading = false;
      })
    },
    //单页条目改变
    sizeChangeHandle: function (val) {
      this.pageSize = val;
      this.pageIndex = 1;
      this.searchUserByPage();
    },
    //翻页
    currentChangeHandle: function (val) {
      this.pageIndex = val;
      this.searchUserByPage();
    },
    //数据查询
    searchHandle: function () {
      this.$refs['dataForm'].validate(valid => {
        if (valid) {
          //清除画面校验信息
          this.$refs['dataForm'].clearValidate();
          //不能向后台传空字符串，但是可以为NULL
          if (this.dataForm.name == '') {
            this.dataForm.name = null;
          }
          //如果当前页面不是第一页则跳转第一页
          if (this.pageIndex != 1) {
            this.pageIndex = 1
          }
          this.searchUserByPage();
        } else {
          return false;
        }
      })
    },
    //新增用户窗口弹出
    addHandle: function () {
      this.addOrUpdateVisible = true;
      this.$nextTick(() => {
        this.$refs.addOrUpdate.init();
      })
    },
    //修改用户窗口弹出
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
          this.$http('user/delete', 'POST', {'ids': ids}, true, res => {
            if (res.num > 0) {
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
  created: function () {
    this.loadRoleList();
    this.loadDeptList();
    this.searchUserByPage();
  }
};
</script>

<style lang="less" scoped="scoped"></style>
