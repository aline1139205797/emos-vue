<template>
  <div>
    <el-form :inline="true" :model="dataForm" :rules="dataRule" ref="dataForm" class="form">
      <el-form-item prop="date">
        <el-date-picker
            v-model="dataForm.date"
            type="date"
            placeholder="选择日期"
            class="input"
            size="medium"
        ></el-date-picker>
      </el-form-item>
      <el-form-item>
        <el-button size="medium" type="primary" @click="searchHandle()">查询</el-button>
        <el-button size="medium" type="danger" @click="addHandle()">会议申请</el-button>
      </el-form-item>
      <el-form-item class="mold">
        <el-radio-group v-model="dataForm.mold" size="medium" @change="changeHandle">
          <el-radio-button label="全部会议"></el-radio-button>
          <el-radio-button label="我的会议"></el-radio-button>
        </el-radio-group>
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
      <el-table-column width="40px" prop="desc" header-align="center" align="center" type="expand">
        <template #default="scope">
          会议内容：{{ scope.row.desc }}
        </template>
      </el-table-column>
      <el-table-column type="index" header-align="center" align="center" width="100" label="序号">
        <template #default="scope">
          <span>{{ (pageIndex - 1) * pageSize + scope.$index + 1 }}</span>
        </template>
      </el-table-column>
      <el-table-column prop="title" header-align="center" align="center" label="会议主题" min-width="400"/>
      <el-table-column prop="creatorName" header-align="center" align="center" min-width="150" label="创建者"/>
      <el-table-column prop="date" header-align="center" align="center" min-width="150" label="日期"/>
      <el-table-column header-align="center" align="center" min-width="150" label="时间">
        <template #default="scope">
          <span>{{ scope.row.start }} ~ {{ scope.row.end }}</span>
        </template>
      </el-table-column>
      <el-table-column prop="num" header-align="center" align="center" min-width="100" label="人数"/>
      <el-table-column prop="status" header-align="center" align="center" min-width="100" label="状态"/>
      <el-table-column header-align="center" align="center" width="150" label="操作">
        <template #default="scope">
          <el-button
              type="text"
              size="medium"
              :disabled="!scope.row.canEnterMeeting"
              @click="enterHandle(scope.row.id, scope.row.uuid)"
          >
            进入
          </el-button>
          <el-button
              type="text"
              size="medium"
              :disabled="
                            !(
                                (scope.row.status == '待审批' || scope.row.status == '未开始') &&
                                scope.row.isCreator == 'true'
                            )
                        "
              @click="deleteHandle(scope.row)"
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
    <add v-if="addVisible" ref="add" @refresh="refresh"></add>
  </div>
</template>

<script>
import Add from './online_meeting-add.vue';
import dayjs from 'dayjs';

export default {
  components: {Add},
  data: function () {
    return {
      dataForm: {
        date: null,
        mold: '全部会议'
      },
      dataList: [],
      pageIndex: 1,
      pageSize: 10,
      totalCount: 0,
      dataListLoading: false,
      addVisible: false,
      dataRule: {}
    };
  },
  methods: {
    //列表初始数据加载
    loadDataList: function () {
      this.dataListLoading = true;
      let data = {
        mold: this.dataForm.mold,
        page: this.pageIndex,
        length: this.pageSize
      }
      if (this.dataForm.date != null && this.dataForm.date != '') {
        data.date = dayjs(this.dataForm.date).format("YYYY-MM-DD");
      }
      this.$http("meeting/searchOnlineMeetingByPage", "POST", data, true, res => {
        for (let obj of res.page.list) {
          switch (obj.status) {
            case 1:
              obj.status = '待审批';
              break;
            case 3:
              obj.status = '未开始';
              break;
            case 4:
              obj.status = '进行中';
              break;
            case 5:
              obj.status = '已结束';
              break;
          }
          //计算会议是否可以进入
          //开会前15分钟可以进入,状态必须是未开始或者进行中
          let minuet = dayjs(new Date()).diff(dayjs(`${obj.date} ${obj.start}`), 'minute');
          if (obj.mine == 'true' && (minuet >= -10 && minuet <= 0 && obj.status == '未开始') || obj.status == '进行中') {
            obj.canEnterMeeting = true;
          } else {
            obj.canEnterMeeting = false;
          }
        }
        this.dataList = res.page.list;
        this.totalCount = res.page.totalCount;
        this.pageSize = res.page.pageSize;
        this.pageIndex = res.page.pageIndex;
        this.dataListLoading = false;
      })
    },
    //改变显示条目
    sizeChangeHandle: function (val) {
      this.pageSize = val;
      this.loadDataList();
    },
    //改变页码
    currentChangeHandle(val) {
      this.pageIndex = val;
      this.loadDataList();
    },
    //表单数据查询
    searchHandle: function () {
      this.$refs['dataForm'].validate(valid => {
        if (valid) {
          this.$refs['dataForm'].clearValidate();
          this.pageIndex = 1;
          this.loadDataList();
        } else {
          return false;
        }
      })
    },
    //会议查询类型变更
    changeHandle: function () {
      this.searchHandle();
    },
    //会议申请
    addHandle: function () {
      this.addVisible = true;
      this.$nextTick(() => {
        this.$refs.add.init();
      })
    },
    //会议申请回调函数
    refresh: function () {
      this.$refs['dataForm'].resetFields();
      this.loadDataList();
    },
    //删除会议申请
    deleteHandle: function (json) {
      this.$confirm('是否删除会议?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        let data = {
          id: json.id,
          uuid: json.uuid,
          instanceId: json.instanceId,
          reason: '删除会议申请'
        };
        this.$http('meeting/DeleteMeetingApplicationForm', 'POST', data, true, res => {
          if (res.row == 1) {
            this.$message({
              message: "操作成功",
              type: "success",
              duration: 1200
            });
            this.searchHandle();
          } else {
            this.$message({
              message: "操作失败",
              type: "error",
              duration: 1200
            });
          }
        })
      })
    },
    //进入会议室
    enterHandle: function (id, uuid) {
      this.$router.push({name: 'MeetingVideo', params: {meetingId: id, uuid: uuid}});
    }
  },
  created: function () {
    this.loadDataList();
  }
};
</script>

<style lang="less" scoped="scoped">
@import url('online_meeting.less');
</style>
