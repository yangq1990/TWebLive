<template>
    <div class="live-pc" >
        <Header  @logout="logout"/>
        <div class="top-box" @click="showMobile">
            <svg t="1591255158096" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="2653" width="32" height="32"><path d="M754.9 64.4H269.1c-10.3 0-18.7 8.4-18.7 18.7v857.8c0 10.3 8.4 18.7 18.7 18.7h485.7c10.3 0 18.7-8.4 18.7-18.7V83.1c0.1-10.3-8.3-18.7-18.6-18.7z m-18.7 37.4v721.6H287.8V101.8h448.4zM475 875.9c0-19.3 15.7-35 35-35s35 15.7 35 35-15.7 35-35 35c-19.3 0.1-35-15.6-35-35z" fill="#ffffff" p-id="2654"></path></svg>
            <span style="color: #ffffff">手机观看</span>
        </div>
<!--        <div>时间</div>-->
        <el-dialog
                title="微信观看"
                :visible.sync="mobileImg"
                width="30%"
                center>
            <img class="mobile-Live" src="../qrcode.png"/>
            <span slot="footer" class="dialog-footer">
            </span>
        </el-dialog>
        <div class="container-pc" >
            <div id="video-container" class="video-container"></div>
            <div id="login-container" v-if="isLogin" >
                <login/>
            </div>
            <div class="chat-container">
                <newChat />
            </div>
        </div>
    </div>


</template>

<script>
  import { mapState } from 'vuex'
  import Vue from 'vue'
  import newChat from './components/message/chat-room'
  import Login from './components/user/login'
  import Header from './components/header'
  import MTA from './utils/mta'
  import {isMobile} from './utils/mobile'
  export default {
    title: 'TWebLive DEMO',
    components: {
      Login,
      Header,
      newChat,
    },
    data() {
      return {
        options: {
          flv: 'https://3891.liveplay.myqcloud.com/live/yqtest.flv',
          m3u8: 'https://3891.liveplay.myqcloud.com/live/yqtest.m3u8',
          autoplay: true,
          width: '100%',
          height: 'auto',
          wording: {
            2032: '请求视频失败，请检查网络',
            2048: '请求m3u8文件失败，可能是网络错误或者跨域问题'
          }
        },
        tweblive:null,
        isJoined:false,
        isMobile:isMobile(),
        mobileImg:false
      }

    },

    computed: {
      ...mapState({
        chatInfo: state => state.conversation.chatInfo,
        currentMessageList: state => state.conversation.currentMessageList,
        userInfo: state => state.user.userInfo,
        isLogin: state => state.user.isLogin,
        isSDKReady: state => state.user.isSDKReady,
        userID: state => state.user.userID
      }),
      // 是否显示 Loading 状态
      showLoading() {
        return !this.isJoined
      }
    },

    mounted() {
      // 初始化监听器
      this.initListener()
    },

    watch: {
      isLogin(next) {
        if (next) {
          MTA.clickStat('link_two', { show: 'true' })
        }
      }
    },

    methods: {
      initListener() {
        const tweblive = new this.TWebLive({
          SDKAppID: 1400187352,
          domID: 'video-container',
          ...this.options
        })
        this.tweblive = tweblive
        window.tweblive = tweblive
        Vue.prototype.tweblive = tweblive
        this.enterRoom()

        // 登录成功后会触发 SDK_READY 事件，该事件触发后，可正常使用 SDK 接口
        this.tweblive.on(this.TWebLive.EVENT.IM_READY, this.onReadyStateUpdate)
        // 被踢出
        this.tweblive.on(this.TWebLive.EVENT.KICKED_OUT, this.onKickedOut)
        // SDK内部出错
        this.tweblive.on(this.TWebLive.EVENT.ERROR, this.onError)
        // 收到自定义新消息
        this.tweblive.on(this.TWebLive.EVENT.CUSTOM_MESSAGE_RECEIVED, this.onCustomMessageReceived)
        // 收到文本新消息
        this.tweblive.on(this.TWebLive.EVENT.TEXT_MESSAGE_RECEIVED, this.onTextMessageReceived)
        // 加入直播间
        this.tweblive.on(this.TWebLive.EVENT.REMOTE_USER_JOIN, this.onRemoteUserJoin)
        // 离开直播间
        this.tweblive.on(this.TWebLive.EVENT.REMOTE_USER_LEAVE, this.onRemoteUserLeave)
        // 网络监测
        this.tweblive.on(this.TWebLive.EVENT.NET_STATE_CHANGE, this.onNetStateChange)
      },
      onTextMessageReceived ({ data: messageList }) {
        messageList.forEach(function(message) {
          const userName = message.nick || message.from
          message.nick = userName
        })
        this.$store.commit('pushCurrentMessageList', messageList)
      },
      onCustomMessageReceived () {
        this.$store.commit('showLike',1)

        // messageList.forEach(() => {
        //   this.$store.commit('showLike',1)
        // })
      },
      onRemoteUserJoin ({ data: messageList }) {
        messageList.forEach(function(message) {
          const userName = message.nick || message.payload.userIDList[0]
          message.payload.text = `${userName} 来了`
          message.type = 'Live-tips'
        })
        if (this.isSDKReady) {
          this.$store.commit('pushCurrentMessageList', messageList)
        }
      },
      onRemoteUserLeave ({ data: messageList }) {
        messageList.forEach(function(message) {
          const userName = message.nick || message.payload.userIDList[0]
          message.payload.text = `${userName} 走了`
          message.type = 'Live-tips'
        })
        this.$store.commit('pushCurrentMessageList', messageList)
      },
      onError({ data }) {
        if (data.message !== '' && data.message !== 'Network Error') {
          // this.$store.commit('showMessage', {
          //   message: data.message,
          //   type: 'error'
          // })
        }

      },
      showMobile() {
        this.mobileImg = true

      },
      onReadyStateUpdate({ name }) {
        const isSDKReady = name === this.TWebLive.EVENT.IM_READY ? true : false
        this.$store.commit('toggleIsSDKReady', isSDKReady)

        if (isSDKReady) {
          this.enterRoom()
          this.getMyProfile()
        }
      },
      // 加入直播间
      enterRoom() {
        this.tweblive.enterRoom(this.chatInfo.groupId).then(() => {
          this.isJoined = true
        })
      },
      exitRoom() {
        this.tweblive.exitRoom(this.chatInfo.groupId).then(() => {
          this.isJoined = false
        })
      },
      getMyProfile() {
        this.tweblive.getMyProfile().then((res) => {
          this.userInfo.nickName = res.data.nick || this.userID
          this.userInfo.avatar = res.data.avatar || this.userInfo.defaultImg
        }).catch(()=>{
          // console.warn('getMyProfile error:', imError)// 更新资料失败的相关信息
        })
      },
      kickedOutReason(type) {
        switch (type) {
          case this.TWebLive.TYPES.KICKED_OUT_MULT_ACCOUNT:
            return '由于多实例登录'
          case this.TWebLive.TYPES.KICKED_OUT_MULT_DEVICE:
            return '由于多设备登录'
          case this.TWebLive.TYPES.KICKED_OUT_USERSIG_EXPIRED:
            return '由于 userSig 过期'
          default:
            return ''
        }
      },
      checkoutNetState(state) {
        switch (state) {
          case this.TWebLive.TYPES.NET_STATE_CONNECTED:
            return { message: '已接入网络', type: 'success' }
          case this.TWebLive.TYPES.NET_STATE_CONNECTING:
            return { message: '当前网络不稳定', type: 'warning' }
          case this.TWebLive.TYPES.NET_STATE_DISCONNECTED:
            return { message: '当前网络不可用', type: 'error' }
          default:
            return ''
        }
      },
      _logout() {
        this.tweblive.logout().then(() => {
          this.$store.commit('toggleIsSDKReady', false)
          // this.$store.commit('reset')
          this.$store.commit('showMessage', { type: 'success', message: '退出成功' })
        })
      },

      async logout() {
        if (this.isSDKReady) {
          await this.exitRoom()
          await this._logout()
          this.enterRoom()
        }
      },
      onNetStateChange(event) {
        this.$store.commit('showMessage', this.checkoutNetState(event.data))
      },
      onKickedOut(event) {
        this.$store.commit('showMessage', {
          message: `${this.kickedOutReason(event.data.type)}被踢出，请重新登录。`,
          type: 'error'
        })
        this.$store.commit('toggleIsLogin', true)
        this.$store.commit('reset')
      }
    }
  }
</script>

<style scoped lang="stylus">
    .live-pc {
        width 90%
        margin 0 auto
        height 32px
        .top-box {
            color #ffffff
            margin 80px 0 20px 0
            cursor pointer
            & span {
                color #E4E4E4
                height 32px
                line-height 32px
                font-size 14px
                vertical-align top
            }
        }
        .container-pc {
            //padding-top 100px    //头部，后续加
            position relative
            display flex
            .video-container {
                float left
                flex 1
                min-width 600px
                height 80vh

            }
            .login-container {
                display flex
                justify-content center
                align-items center
                flex-direction column
                //padding-top: 100px;
            }
            .chat-container{
                position relative
                width 360px
                background #141520
                color #ffffff
                margin-left 10px
                border-radius 5px 5px 0 0
            }

        }


    }
    //手机二维码
    .el-dialog__wrapper {
        display flex
        justify-content center
        align-items center
    }
    /deep/ .el-dialog--center {
        margin 0 !important
        width 360px !important
        padding 5px 0
    }
    /deep/ .el-dialog__title {
        color #1A8CFF
        font-size 16px
    }
    .mobile-Live {
        display block
        margin 0 auto
        /*width 60px*/
        /*height 60px*/
    }

    //修改视频样式
    /deep/ .vcp-player video {
        position absolute
        top 0
        left 0
        width 100%
        height 100%
    }
    /deep/ .vcp-player {
        width 100%
        height 100%
    }
    /deep/ .vcp-bigplay{
        display none
    }



    @media screen  and (max-width:1000px){
        .container-pc {
            overflow-x auto
        }
    }
  @media screen and (min-width: 1200px) and (max-width:1500px){
      .live-pc {
          width 85%
      }
  }
  @media screen and (min-width: 1500px){
      .live-pc {
          width 70%
      }
  }

</style>
