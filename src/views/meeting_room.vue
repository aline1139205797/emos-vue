<template>
  <div>
    <el-form :inline="true" :model="dataForm" :rules="dataRule" ref="dataForm">
      <el-form-item prop="name">
        <el-input
            v-model="dataForm.name"
            placeholder="会议室名称"
            size="medium"
            class="input"
            clearable="clearable"
        />
      </el-form-item>
      <el-form-item>
        <el-select v-model="dataForm.canDelete" class="input" placeholder="条件" size="medium">
          <el-option label="全部" value="all"/>
          <el-option label="可删除" value="true"/>
          <el-option label="不可删除" value="false"/>
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button size="medium" type="primary" @click="searchHandle()">查询</el-button>
        <el-button
            size="medium"
            type="primary"
            :disabled="!isAuth(['ROOT', 'MEETING_ROOM:INSERT'])"
            @click="addHandle()"
        >
          新增
        </el-button>
        <el-button
            size="medium"
            type="danger"
            :disabled="!isAuth(['ROOT', 'MEETING_ROOM:DELETE'])"
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
      <el-table-column prop="name" header-align="center" align="center" min-width="150" label="会议室名称"/>
      <el-table-column header-align="center" align="center" min-width="120" label="人数上限">
        <template #default="scope">
          <span>{{ scope.row.max }}人</span>
        </template>
      </el-table-column>
      <el-table-column header-align="center" align="center" min-width="100" label="状态">
        <template #default="scope">
          <span>{{ scope.row.status == 1 ? '可使用' : '已停用' }}</span>
        </template>
      </el-table-column>
      <el-table-column prop="desc" header-align="center" align="center" label="备注" min-width="400"/>
      <el-table-column header-align="center" align="center" width="150" label="操作">
        <template #default="scope">
          <el-button
              type="text"
              size="medium"
              :disabled="!isAuth(['ROOT', 'MEETING_ROOM:UPDATE']) || scope.row.id == 0"
              @click="updateHandle(scope.row.id)"
          >
            修改
          </el-button>
          <el-button
              type="text"
              size="medium"
              :disabled="!isAuth(['ROOT', 'MEETING_ROOM:DELETE']) || scope.row.id == 0"
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
import AddOrUpdate from './meeting_room-add-or-update.vue';

export default {
  components: {
    AddOrUpdate
  },
  data: function () {
    return {
      dataForm: {
        name: null,
        canDelete: null
      },
      dataList: [],
      pageIndex: 1,
      pageSize: 10,
      totalCount: 0,
      dataListLoading: false,
      dataListSelections: [],
      addOrUpdateVisible: false,
      dataRule: {
        name: [{required: false, pattern: '^[a-zA-Z0-9\u4e00-\u9fa5]{1,20}$', message: '会议室名称格式错误'}]
      }
    };
  },
  methods: {

    selectionChangeHandle: function (val) {
      this.dataListSelections = val;
    },
    sizeChangeHandle: function (val) {
      this.pageSize = val;
      this.pageIndex = 1;
      this.loadDataList();
    },
    currentChangeHandle: function (val) {
      this.pageIndex = val;
      this.loadDataList();
    },
    //分页数据初始化
    loadDataList: function () {
      this.dataListLoading = true;
      let date = {
        "name": this.dataForm.name,
        "canDelete": this.dataForm.canDelete == "all" ? null : this.dataForm.canDelete,
        "pageIndex": this.pageIndex,
        "pageSize": this.pageSize
      }
      this.$http("meeting_room/SearchMeetingRoomByPage", "POST", date, true, res => {
        this.dataList = res.page.list;
        this.totalCount = res.page.totalCount;
        this.dataListLoading = false;
      })
    },
    searchHandle: function () {
      this.$refs['dataForm'].validate(valid => {
        if (valid) {
          this.$refs['dataForm'].clearValidate();
          if (this.dataForm.name == "") {
            this.dataForm.name = null;
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
    //新增会议室弹窗
    addHandle: function () {
      this.addOrUpdateVisible = true;
      this.$nextTick(() => {
        this.$refs.addOrUpdate.init();
      })
    },
    //修改会议室弹窗
    updateHandle: function (id) {
      this.addOrUpdateVisible = true;
      this.$nextTick(() => {
        this.$refs.addOrUpdate.init(id);
      })
    },
    //删除会议室信息
    deleteHandle: function (id) {
      let ids = id ? [id] : this.dataListSelections.map(item => {
        return item.id;
      });
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
          this.$http("meeting_room/deleteByIds", "POST", {"ids": ids}, true, res => {
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

<style lang="less"></style>
