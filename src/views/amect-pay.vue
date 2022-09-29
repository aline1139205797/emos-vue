<template>
  <el-dialog title="缴纳罚款" :close-on-click-modal="false" v-model="visible" width="305px" center>
    <img :src="qrCode" class="qrcode" v-if="!result"/>
    <div v-if="result" class="pay-success">
      <el-result icon="success" title="付款成功" subTitle="请根据提示进行操作"></el-result>
    </div>
    <div class="dialog-footer-style">
      <el-button type="danger" size="medium" v-if="!result" @click="closeHandle">取消支付</el-button>
      <el-button type="primary" size="medium" v-if="!result" @click="successHandle">支付成功</el-button>
      <el-button type="primary" size="medium" v-if="result" @click="closeHandle">关闭窗口</el-button>
    </div>
  </el-dialog>
</template>

<script>
export default {
  data: function () {
    return {
      visible: false,
      dataForm: {
        id: null
      },
      result: false,
      qrCode: null
    };
  },
  methods: {
    init: function (id) {
      this.visible = true;
      this.dataForm.id = id;
      this.result = false;
      this.$nextTick(() => {
        //从cookie中过去token令牌
        let token = this.$cookies.get('token');
        //向后台socket服务类发起请求,缓存token
        this.$socket.sendObj({opt: "pay_amect", token: token});
        //监听后台推送消息
        this.$options.sockets.onmessage = res => {
          if (res.data == '付款成功') {
            this.result = true;
          }
        };
        this.$http('amect/CreateNativeAmectPayOrder', 'POST', {amectId: id}, true, res => {
          this.qrCode = res.qrCodeBase64;
        })
      })
    },
    //关闭窗口
    closeHandle: function () {
      this.visible = false;
      this.$emit('refreshDataList');
    },
    //查询支付结果
    successHandle: function () {
      this.visible = false;
      this.$http('amect/SearchNativeAmectPayResult', 'POST', {amectId: this.dataForm.id}, true, res => {
        this.$emit('refreshDataList');
      })
    }
  }
};
</script>

<style scoped>
.qrCode {
  width: 255px;
  height: 255px;
}

.pay-success {
  width: 255px;
  height: 255px;
}

.dialog-footer-style {
  padding-bottom: 30px;
  padding-top: 10px;
  text-align: center;
}
</style>
