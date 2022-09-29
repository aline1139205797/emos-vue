<template>
  <el-dialog title="执行归档" width="500px" :close-on-click-modal="false" v-model="visible" :show-close="false">
    <el-upload
        ref="upload"
        :action="url"
        list-type="picture-card"
        accept=".jpg,.jpeg,.png"
        with-credentials="true"
        :before-upload="beforeUploadHandle"
        :on-success="successHandle"
        :on-remove="removeHandle"
    >
      <i class="el-icon-plus"></i>
    </el-upload>
    <template #footer>
			<span class="dialog-footer">
				<el-button size="medium" @click="cancel()">取消</el-button>
				<el-button type="primary" @click="archive()" size="medium" :disabled="disableBtn">{{ btn }}</el-button>
			</span>
    </template>
  </el-dialog>
</template>

<script>
export default {
  data: function () {
    return {
      visible: false,
      url: this.$baseUrl + 'cos/uploadCosFile?type=archive',
      taskId: '',
      picList: {},
      disableBtn: false,
      btn: '确定'
    };
  },
  methods: {
    //页面初始化
    init: function (id) {
      this.visible = true;
      this.taskId = id;
      this.btn = '执行归档';
      this.disableBtn = false;
      this.picList = {};
      this.$nextTick(() => {
        this.$refs['upload'].clearFiles();
      })
    },
    //关闭窗口
    cancel: function () {
      if (Object.keys(this.picList).length > 0) {
        let pathes = Object.values(this.picList);
        console.log(this.picList);
        console.log("====================>")
        console.log(pathes);
        this.$http('cos/deleteCosFile', 'POST', {pathes: pathes}, true, res => {
          this.picList = {};
        });
      }
      this.visible = false;
      this.$refs['upload'].clearFiles();
    },
    //文件上传前事件
    beforeUploadHandle: function (file) {
      if (file.type !== 'image/jpg' && file.type !== 'image/jpeg' && file.type !== 'image/png') {
        this.$message.error('只支持jpg、png格式的图片！');
        return false;
      }
      return true;
    },
    //文件上传完毕回调函数
    successHandle: function (resp, file, fileList) {
      if (resp && resp.code === 200) {
        for (let one of fileList) {
          this.picList[one.response.url] = one.response.path;
        }
      } else {
        this.$message.error('图片上传失败');
      }
    },
    //删除图片
    removeHandle: function (file, fileList) {
      let url = file.response.url;
      let path = this.picList[url];
      this.$http('cos/deleteCosFile', 'POST', {pathes: [path]}, true, res => {
        delete this.picList[url];
      });
    },
    //归档
    archive:function (){
      this.btn = '正在归档';
      this.disableBtn = true;
      if(!Object.keys(this.picList).length > 0){
        this.$message({
          message: '没有上传归档照片',
          type: 'error',
          duration: 1200
        });
        this.btn = '执行归档';
        this.disableBtn = false;
        return;
      };
      let files = [];
      for(let key in this.picList){
        files.push({
          url: key,
          path: this.picList[key]
        });
      }
      let data = {
        taskId: this.taskId,
        files: JSON.stringify(files)
      }
      this.$http('approval/archiveTask','POST',data,true,res => {
        this.$message({
          message: '操作成功',
          type: 'success',
          duration: 1200
        });
        this.visible = false;
        this.$emit('refreshDataList');
      })
    }

  }
};
</script>

<style lang="less"></style>
