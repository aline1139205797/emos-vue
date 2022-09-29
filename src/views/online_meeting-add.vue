<template>
  <el-dialog title="线上会议申请" :close-on-click-modal="false" v-model="visible" width="692px">
    <el-form :model="dataForm" ref="dataForm" :rules="dataRule" label-width="60px">
      <el-form-item label="主题" prop="title">
        <el-input v-model="dataForm.title" size="medium" style="width:100%" clearable="clearable"/>
      </el-form-item>
      <el-form-item label="内容" prop="desc">
        <el-input
            type="textarea"
            :rows="2"
            placeholder="请输入内容"
            v-model="dataForm.desc"
            size="medium"
            resize="none"
            maxlength="200"
            clearable="clearable"
        />
      </el-form-item>
      <el-form-item label="日期" prop="date">
        <el-date-picker
            v-model="dataForm.date"
            type="date"
            placeholder="选择日期"
            style="width:34.5%"
            size="medium"
            :disabledDate="disabledDate"
            clearable="clearable"
            @change="checkTime()"
        ></el-date-picker>
        <span class="note">会议日期只能是当前或者未来的日期，不能是以往的日期</span>
      </el-form-item>
      <el-form-item label="时间">
        <el-col :span="11">
          <el-form-item prop="start" class="inner-item">
            <el-time-select
                placeholder="起始时间"
                v-model="dataForm.start"
                start="08:30"
                step="00:30"
                end="18:30"
                size="medium"
                style="width:96%"
                clearable="clearable"
                :minTime="nowDate"
                @change="checkTime()"
            ></el-time-select>
          </el-form-item>
        </el-col>
        <el-col :span="2">&nbsp;&nbsp;~&nbsp;&nbsp;</el-col>
        <el-col :span="11" prop="end">
          <el-form-item prop="end" class="inner-item">
            <el-time-select
                placeholder="结束时间"
                v-model="dataForm.end"
                start="08:30"
                step="00:30"
                end="18:30"
                size="medium"
                style="width:96%"
                clearable="clearable"
                :minTime="dataForm.start"
                @change="checkTime()"
            ></el-time-select>
          </el-form-item>
        </el-col>
        <span class="note">注意会议时间范围</span>
      </el-form-item>
      <el-form-item label="成员" prop="members">
        <el-transfer
            v-model="dataForm.members"
            :data="users"
            :titles="['员工', '参会人']"
            filterable
            filter-placeholder="请输入姓名"
        />
      </el-form-item>
    </el-form>
    <template #footer>
			<span class="dialog-footer">
				<el-button size="medium" @click="visible = false">取消</el-button>
				<el-button type="primary" size="medium" @click="dataFormSubmit">确定</el-button>
			</span>
    </template>
  </el-dialog>
</template>

<script>
import dayjs from "dayjs";

export default {
  data: function () {
    return {
      visible: false,
      nowDate: dayjs(new Date()).format("HH:mm"),
      dataForm: {
        title: null,
        date: null,
        start: null,
        end: null,
        desc: null,
        members: [],
        type: 1
      },
      disabledDate(date) {
        return date.getTime() < Date.now() - 24 * 60 * 60 * 1000;
      },
      users: [],
      dataRule: {
        title: [{required: true, pattern: '^[a-zA-Z0-9\u4e00-\u9fa5]{2,30}$', message: '会议主题格式错误'}],
        desc: [{required: true, message: '会议内容为必填'}],
        date: [{required: true, message: '日期为必填'}],
        start: [{required: true, message: '起始时间为必填'}],
        end: [{required: true, message: '结束时间为必填'}],
        members: [
          {required: true, trigger: 'blur', message: '必须设置参会人'},
          {required: false, trigger: 'change', message: '必须设置参会人'}
        ]
      }
    };
  },
  methods: {
    init: function (id) {
      let that = this;
      that.visible = true;
      that.$nextTick(() => {
        that.$refs['dataForm'].resetFields();
        that.$http('user/searchAllUser', 'GET', null, true, function (resp) {
          let temp = [];
          for (let one of resp.list) {
            temp.push({key: one.id, label: one.name});
          }
          that.users = temp;
        });
      });
    },
    //申请提交
    dataFormSubmit: function () {
      this.$refs['dataForm'].validate(valid => {
        let data = JSON.parse(JSON.stringify(this.dataForm));
        data.date = dayjs(this.dataForm.date).format("YYYY-MM-DD");
        data.members = JSON.stringify(this.dataForm.members);
        if (valid) {
          this.$refs['dataForm'].clearValidate();
          this.$http('meeting/insertMeeting', 'POST', data, true, res => {
            if (res.row == 1){
              this.visible = false;
              this.$message({
                message: "操作成功",
                type: "success",
                duration: 1200
              });
              setTimeout(() => {
                this.$emit('refresh'); //刷新数据
              }, 1200);
            }else{
              this.$message({
                message: "操作失败",
                type: "error",
                duration: 1200
              });
            }
          })
        } else {
          return false;
        }
      })
    },
    //会议室预约时间校验
    checkTime: function () {
      if (this.dataForm.date == null || this.dataForm.date == '') {
        return false;
      } else {
        if (!dayjs().isAfter(dayjs(this.dataForm.date))) {
          this.nowDate = null;
        } else {
          this.nowDate = dayjs(new Date()).format("HH:mm");
        }
      }
      if (this.dataForm.start == null || this.dataForm.start == '') {
        return false;
      } else {
        let nowDay = dayjs(new Date()).format("YYYY-MM-DD");
        if (this.nowDate != null && dayjs(nowDay + " " + this.dataForm.start).isBefore(dayjs(nowDay + " " + this.nowDate))) {
          this.dataForm.start = null;
          this.dataForm.end = null;
          return false;
        }
        if (!dayjs(nowDay + " " + this.dataForm.start).isBefore(dayjs(nowDay + " " + this.dataForm.end))) {
          this.dataForm.end = null;
          return false;
        }
      }
      if (this.dataForm.end == null || this.dataForm.end == '') {
        return false;
      }
      return true;
    },
  }
};
</script>

<style lang="less" scoped="scoped">
</style>
