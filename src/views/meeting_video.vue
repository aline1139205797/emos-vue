<template>
  <div>
    <el-row :gutter="15">
      <!-- 视频墙 -->
      <el-col :span="19">
        <div class="meeting-container">
          <el-scrollbar height="650px" id="videoListContainer">
            <div class="video-list">
              <div class="video">
                <div class="user">
                  <img class="photo" :src="mine.photo"/>
                  <span class="name">{{ mine.name }}{{ shareStatus ? '（共享中）' : '' }}</span>
                </div>
                <div id="localStream" @dblclick="bigVideoHandle('localStream')"></div>
              </div>
              <div class="video" v-for="one in memberList">
                <div class="user">
                  <img class="photo" :src="one.photo"/>
                  <span class="name">{{ one.name }}</span>
                </div>
                <div :id="one.id" class="remote-stream" @dblclick="bigVideoHandle(one.id)"></div>
              </div>
            </div>
          </el-scrollbar>
          <div id="videoBig" @dblclick="smallVideoHandle()"></div>
        </div>
        <p class="desc">
          会议过程中，不需要发言的会场要主动将本地会场的MIC关闭，保证会场安静，当需要发言时要及时打开MIC。会议过程中，需要发言讨论时，先打开MIC向主会场提出请求，得到同意后再继续发言，否则请继续保持静音。发言时，要—个人一个人的发言，不要多人同时讲话，因为全向MIC会把所有人的声音混合，远端听到的声音会非常嘈杂，听不清具体说话内容。在会议进行过程中，尽量控制会场噪音，不要在会场中随意走动
        </p>
      </el-col>
      <!-- 用户列表区域 -->
      <el-col :span="5">
        <el-card shadow="never">
          <template #header>
            <div class="card-header"><span>用户列表</span></div>
          </template>
          <el-scrollbar height="555px">
            <ul class="user-list">
              <li v-for="one in userList">
                <img class="mic" src="../assets/trtc/mic.png"/>
                <div class="mic-container">
                  <img
                      class="mic-green"
                      :id="'mic-' + one.userId"
                      src="../assets/trtc/mic-green.png"
                  />
                </div>
                <span>{{ one.dept }} - {{ one.name }}</span>
              </li>
            </ul>
          </el-scrollbar>
        </el-card>
        <div class="meeting-operate">
          <button :class="meetingStatus ? 'phone-btn-off' : 'phone-btn-on'" @click="phoneHandle"></button>
          <button :class="videoStatus ? 'video-btn-on' : 'video-btn-off'" @click="videoHandle"></button>
          <button :class="micStatus ? 'mic-btn-on' : 'mic-btn-off'" @click="micHandle"></button>
          <button :class="shareStatus ? 'share-btn-on' : 'share-btn-off'" @click="shareHandle"></button>
        </div>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import TRTC from 'trtc-js-sdk';
import $ from 'jquery';

export default {
  data: function () {
    return {
      meetingId: null,
      uuid: null,
      appId: null,
      userSign: null,
      userId: null,
      roomId: null,
      meetingStatus: false,
      videoStatus: true,
      micStatus: true,
      shareStatus: false,
      userList: [], //进入会场的用户列表
      mine: {},
      memberList: [], //会议成员列表
      client: null,
      localStream: null,
      shareStream: null,
      stream: {}, //所有的远端流
      bigVideoUserId: null //大屏显示远端流的用户ID，切换回小屏幕的时候使用
    };
  },
  methods: {
    phoneHandle: function () {
      //检测浏览器是否支持在线会议
      TRTC.checkSystemRequirements().then(check => {
        if (!check.result) {
          this.$alert('当前浏览器不支持在线会议', '提示', {
            confirmButtonText: '确定'
          });
        } else {
          this.meetingStatus = !this.meetingStatus;
          if (this.meetingStatus) {
            //记录摄像头与话筒状态
            this.videoStatus = true;
            this.micStatus = true;
            //TRTC日志输出级别
            TRTC.Logger.setLogLevel(TRTC.Logger.LogLevel.ERROR);

            //生成用户签名
            this.$http('/meeting/searchMyUserSig', 'GET', {}, false, res => {
              this.appId = res.appId;
              this.userSign = res.userSig;
              this.userId = res.userId;
            });

            //创建TRTC Client对象
            let client = TRTC.createClient({
              mode: 'rtc',
              sdkAppId: this.appId,
              userId: this.userId,
              userSig: this.userSign
            });
            this.client = client;

            //监听远端流事件(用户进入会议室时触发)
            client.on('stream-added', event => {
              let remoteStream = event.stream;
              //订阅远端流
              client.subscribe(remoteStream);
              //从远端流获取用户ID
              let userId = remoteStream.getUserId();
              //把新上线的用户添加到右侧在线人员名单
              this.$http('/user/SearchNameAndDept', 'POST', {id: userId}, true, res => {
                this.userList.push({
                  userId: userId,
                  name: res.name,
                  dept: res.dept
                });
              })
              //把远端流保存起来,大屏显示某个远端流时用到
              this.stream[userId] = remoteStream;
            });

            //远端流订阅成功事件
            client.on('stream-subscribe', event => {
              let remoteStream = event.stream;
              let userId = remoteStream.getUserId();
              $('#' + userId).css({'z-index': 1});
              //播放远端视频流
              remoteStream.play(userId + '');
            });

            //订阅远端删除流事件(用户退出会议室)
            client.on('stream-removed', event => {
              let remoteStream = event.stream;
              //取消订阅远端流
              client.unsubscribe(remoteStream);
              let userId = remoteStream.getUserId();
              //把下线的用户从右侧在线人员名单删除
              let i = this.userList.findIndex(obj => {
                return userId == obj;
              });
              this.userList.splice(i, 1);

              //停止播放远端流视频并关闭远端流
              remoteStream.stop();
              remoteStream.close();

              //删除保存的远端流,并把视频窗口置底,显示用户基本信息
              delete this.stream[userId];
              $('#' + userId).css({'z-index': '-1'});
              $('#' + userId).html('');
            });

            //订阅语音事件(远端本地都会触发)
            client.on('audio-volume', event => {
              event.result.forEach(({userId, audioVolume, stream}) => {
                //声音大于5就设置动画
                if (audioVolume > 5) {
                  $('#mic-' + userId).css('top', `${100 - audioVolume * 3}%`)
                } else {
                  $('#mic-' + userId).css('top', `100%`)
                }
              })
            });
            //开启音量回调函数并设置30ms触发一次
            client.enableAudioVolumeEvaluation(30);

            client.join({roomId: this.roomId}).then(() => {
              //会议签到
              this.$http('/meeting/UpdateMeetingPresent', 'POST', {meetingId: this.meetingId}, true, res => {
                if (res.row == 1) {
                  this.$message({
                    message: '签到成功',
                    type: 'success',
                    time: 1200
                  })
                }
              })

              //成功进入会议室,创建本地音视频流
              let localStream = TRTC.createStream({
                userId: this.userId + '',
                audio: true,
                video: true
              });
              this.localStream = localStream;
              //设置分辨率
              localStream.setVideoProfile('480P');

              //把自己添加到用户列表中
              this.$http('/user/SearchNameAndDept', 'POST', {id: this.userId}, true, res => {
                this.userList.push({
                  userId: this.userId,
                  name: res.name,
                  dept: res.dept
                });
              })
              //初始化本地音视频流
              localStream.initialize().catch(error => {
                console.error('初始化本地流失败' + error);
              }).then(() => {
                console.log('创建本地视频流成功');
                //视频窗口置顶,播放本地视频流
                $('#localStream').css({'z-index': 1});
                localStream.play('localStream');
                //向远端用户推送视频流
                client.publish(localStream).catch(error => {
                  console.error('本地视频流发布失败' + error);
                }).then(() => {
                  console.log('本地视频流发布成功');
                })
              })
            }).catch(error => {
              console.error('进入房间失败' + error);
            })

          } else {
            //关闭视频会议
            let stream = this.getStream();
            this.client.unpublish(stream).then(() => {
              this.client.leave().then(() => {
                console.log("成功退出会议室");
                //关闭本地流
                stream.stop();
                stream.close();
                //清空模型层本地流
                this.shareStream = null;
                this.localStream = null;
                //清空模型层远端流
                this.stream = null;
                //销毁TRTC Client对象
                this.client = null;
                //清空用户列表
                this.userList = null;
                this.videoStatus = true;
                this.micStatus = true;
                this.shareStatus = false;
                //视频墙DIV置底
                $('#localStream').css('z-index', '-1');
                $('#localStream').html('');
                //大屏播放时隐藏大屏
                if (this.bigVideoUserId != null) {
                  $('#videoListContainer').show();
                  $('videoBig').hide();
                  this.bigVideoUserId = null;
                }
              }).catch(error => {
                console.error('退出会议室失败' + error);
              })
            })

          }
        }
      })
    },
    //判断使用的是本地流还是共享流,并返回流对象
    getStream: function () {
      let stream = null;
      if (this.shareStream != null) {
        stream = this.shareStream;
      } else if (this.localStream != null) {
        stream = this.localStream;
      }
      return stream;
    },
    //切换大屏显示
    bigVideoHandle: function (userId) {
      this.bigVideoUserId = userId;
      $('#videoListContainer').hide();
      $('#videoBig').show();
      /*this.stream[userId].stop();
      this.stream[userId].play('videoBig');*/
      this.localStream.stop();
      this.localStream.play('videoBig');
    },
    //关闭大屏,恢复小屏显示
    smallVideoHandle: function () {
      /*this.stream[this.bigVideoUserId].stop();
      this.stream[this.bigVideoUserId].play(this.bigVideoUserId);*/
      this.localStream.stop();
      this.localStream.play(this.bigVideoUserId);
      this.bigVideoUserId = null;
      $('#videoListContainer').show();
      $('#videoBig').hide();
    },
    //关闭摄像头
    videoHandle: function () {
      if (this.shareStatus) {
        this.$alert('屏幕共享中无法开关摄像头', '提示信息', {
          confirmButtonText: '确定'
        });
        return;
      }
      if (this.videoStatus) {
        //关闭摄像头
        this.localStream.muteVideo();
      } else {
        //开启摄像头
        this.localStream.unmuteVideo();
      }
      this.videoStatus = !this.videoStatus;
    },
    micHandle: function () {
      let stream = this.getStream();
      if (this.micStatus) {
        //关闭麦克风
        stream.muteAudio();
      } else {
        //开启麦克风
        stream.unmuteAudio();
      }
      this.micStatus = !this.micStatus;
    },
    shareHandle: function () {
      if (!this.meetingStatus) {
        this.$alert('请先进入会议室', '提示信息', {
          confirmButtonText: '确定'
        })
        return;
      }
      //检查浏览器是否支持屏幕共享
      if (!TRTC.isScreenShareSupported()) {
        this.$alert('当前浏览器不支持屏幕共享', '提示信息', {
          confirmButtonText: '确定'
        })
        return;
      }
      this.shareStatus = !this.shareStatus;
      //开启屏幕共享
      if (this.shareStatus) {
        //创建共享流
        let shareStream = TRTC.createStream({
          audio: this.micStatus,
          screen: true,
          userId: this.userId
        })
        shareStream.setScreenProfile('1080p');
        this.shareStream = shareStream;
        shareStream.initialize().catch(error => {
          console.error('创建共享流失败' + error)
        }).then(() => {
          //取消推送本地流
          this.client.unpublish(this.localStream).then(() => {
            this.localStream.close(); //关闭本地流
            this.localStream = null; //初始化本地流
            //视频墙DIV置底
            $('#localStream').css('z-index', '-1');
            this.client.publish(shareStream); //向远端推送共享流
            //shareStream.play('localStream');
          })
        })
      }
      //关闭屏幕共享
      else {
        //重建本地流
        let localStream = TRTC.createStream({
          userId: this.userId + '',
          audio: true,
          video: true
        });
        this.localStream = localStream;
        //设置分辨率
        localStream.setVideoProfile('480P');
        //初始化本地音视频流
        localStream.initialize().catch(error => {
          console.error('初始化本地流失败' + error);
        }).then(() => {
          console.log('创建本地视频流成功');
          this.client.unpublish(this.shareStream).then(() => {
            this.shareStream.close();
            this.shareStream = null;
            //视频窗口置顶,播放本地视频流
            $('#localStream').css({'z-index': 1});
            localStream.play('localStream');
            //向远端用户推送视频流
            this.client.publish(localStream).catch(error => {
              console.error('本地视频流发布失败' + error);
            }).then(() => {
              console.log('本地视频流发布成功');
            })
          })
        })
      }
    }
  },
  created: function () {
    //获取会议人员列表
    let params = this.$route.params;
    this.meetingId = params.meetingId;
    this.uuid = params.uuid;
    let data = {meetingId: this.meetingId};
    this.$http('/meeting/searchOnlineMeetingMembers', 'POST', data, true, res => {
      if (res.list != null && res.list.length != 0) {
        this.mine = res.list[0];
        for (let i = 1; i < res.list.length; i++) {
          this.memberList.push(res.list[i]);
        }
        console.log(this.memberList);
      }
    });

    data = {uuid: this.uuid};
    this.$http('/meeting/searchRoomIdByUUID', 'POST', data, true, res => {
      if (res.roomId == null) {
        this.$message({
          message: '不存在视频会议室',
          type: 'error',
          duration: 1200
        });
      } else {
        this.roomId = res.roomId;
      }
    })
  }
}
</script>

<style lang="less" scoped="scoped">
@import url('meeting_video.less');
</style>
