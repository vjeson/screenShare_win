<template>
  <div id="app">
    <div class="app_title">
      <div class="app_name">屏幕共享</div>
      <div class="app_control">
        <div class="app_control_btn">
          <img @click="window_min" title="最小化" src="~@/assets/img/min.png">
        </div>
        <div class="app_control_btn">
          <img @click="window_close" title="关闭" src="~@/assets/img/close.png">
        </div>
      </div>
    </div>
    <div class="app_container">
      <div class="list_panel">
        <div class="title">
          <span class="text">请选择设备</span>
          <span class="refresh" @click="refresh" title="刷新设备列表">刷新</span>
        </div>
        <div class="list">
          <div class="item" :class="{selected:selectClientId===client.ip}" v-for="client in clientList">
            <div class="text" @click="selectClientId=client.ip">{{client.name}}</div>
          </div>
        </div>
      </div>
      <div class="list_panel">
        <div class="title">
          <span class="text">请选择屏幕</span>
        </div>
        <div class="list">
          <div class="item" :class="{selected:selectScreenIndex===index}"
               v-for="(screen,index) in screenList">
            <div class="text" @click="selectScreenIndex=index">屏幕{{index+1}}</div>
          </div>
        </div>
      </div>
    </div>
    <div class="btn_box">
      <button @click="connectUser()">连接</button>
    </div>

    <div ref="dialog" class="dialog" v-if="dialogOption.show">
      <div class="dialog_container">
        <div class="title" v-if="dialogOption.title">{{dialogOption.title}}</div>
        <div class="msg">{{dialogOption.msg}}</div>
        <div class="buttons">
          <button @click="eventBus.emit('dialogCancel')" v-if="dialogOption.showCancelButton">
            {{dialogOption.cancelButtonText}}
          </button>
          <button @click="eventBus.emit('dialogConfirm')">{{dialogOption.confirmButtonText}}</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
  import {screen, remote} from 'electron'

  const EventEmitter = require('events').EventEmitter;

  const http = require('http');
  const server = http.createServer();
  server.listen(3000);
  const io = require('socket.io')(server);

  const dgram = require('dgram');
  const udp4 = dgram.createSocket('udp4');

  const offerOptions = {
    offerToRecceiveAudio: 1,
    offerToReceiveVideo: 1
  };
  const configuration = {
    iceServers: [{url: "stun:stun.l.google.com:19302"}]
  };
  const constraints = {
    video: {
      mandatory: {
        chromeMediaSource: 'screen',
        // chromeMediaSource: 'desktop',
        // chromeMediaSourceId: '',
        // maxWidth: 1440,
        // maxHeight: 900
        // frameRate: {ideal: 10, max: 15}
      },
      // optional: [
      //   {minFrameRate: 100},
      //   {maxWidth: 10000},
      //   {maxHeigth: 1000}
      // ]
      // height: 300
    },
  };
  export default {
    name: 'screen_share',
    data() {
      return {
        clientList: [],
        screenList: [],
        selectClientId: '',
        selectScreenIndex: 0,

        showDialog: false,
        dialogOption: {
          show: false,
          title: "",
          msg: '',
          confirmButtonText: '',
          cancelButtonText: '',
          showCancelButton: false,
        },
        eventBus: new EventEmitter(),
      }
    },
    mounted() {
      this.startUdp();

      io.on('connect', socket => {
        console.log('已连接');
        let socketId = socket.id;

        socket.emit("message", {type: "init"});

        let pc = new RTCPeerConnection(configuration);

        pc.onicecandidate = (evt) => {
          if (!evt.candidate) return;
          console.log('发送一个candidate', evt.candidate);
          //需要发送到其他客户端
          socket.emit('message', {
            type: 'candidate',
            payload: {
              id: evt.candidate.sdpMid,
              label: evt.candidate.sdpMLineIndex,
              candidate: evt.candidate.candidate,
            },
          });
        };

        socket.on('test', (arg) => {
          console.log('测试', arg);
        });

        socket.on('message', details => {
          console.log("handleMessage", details)

          let type = details.type;
          let payload = details.payload;

          if (type === "init") {

          } else if (type === 'offer') {
            pc.setRemoteDescription(new RTCSessionDescription(payload));
            //进行连接
            navigator.mediaDevices.getUserMedia(constraints)
              .then((mediaStream) => {
                console.log('初始化视频源');

                pc.addStream(mediaStream);
                // selfVideo.srcObject = mediaStream;
                pc.createAnswer(offerOptions)
                  .then(function (answer) {
                    console.log('发送一个anwser');
                    pc.setLocalDescription(answer);
                    socket.emit("message", {
                      payload: answer,
                      type: "answer"
                    });
                  }).catch(function (err) {
                  console.error(err);
                });
              })
              .catch(function (err) {
                /* handle the error */
                console.log(err)
              });


          } else if (type === 'answer') {

            pc.setRemoteDescription(new RTCSessionDescription(payload));

          } else if (type === 'candidate') {
            pc.addIceCandidate(new RTCIceCandidate({
              sdpMLineIndex: payload.label,
              sdpMid: payload.id,
              candidate: payload.candidate
            }));

          }

        });


        socket.on('disconnect', () => {
          console.log('已断开')

        });
      });

      let screens = screen.getAllDisplays();

      let videoWidth = 0;
      let videoHeight = 0;
      for (let i = 0; i < screens.length; i++) {
        let _screen = screens[i].bounds;
        this.screenList.push({
          id: screens[i].id,
          ..._screen
        });
        videoWidth = Math.max(videoWidth, (_screen.width + _screen.x));
        videoHeight = Math.max(videoHeight, (_screen.height + _screen.y));
      }
      this.screenList.forEach(item => {
        item.allWidth = videoWidth;
        item.allHeight = videoHeight;
      });

    },
    methods: {
      messageBox(message, title, option = {}) {
        this.dialogOption.title = title;
        this.dialogOption.msg = message;
        this.dialogOption.show = true;

        this.dialogOption.confirmButtonText = option.confirmButtonText || "确定";
        this.dialogOption.cancelButtonText = option.cancelButtonText || "取消";
        this.dialogOption.showCancelButton = option.showCancelButton || false;

        return new Promise((resolve, reject) => {
          this.eventBus.once("dialogConfirm", () => {
            this.dialogOption.show = false;
            this.eventBus.removeAllListeners("dialogCancel");
            resolve();
          });
          this.eventBus.once("dialogCancel", () => {
            this.dialogOption.show = false;
            this.eventBus.removeAllListeners("dialogConfirm");
            reject("cancel");
          });
        })
      },
      connectUser() {
        if (!this.selectClientId) return this.messageBox("请选择设备！", "提示");
        let client = this.clientList.find(item => item.ip === this.selectClientId);
        let screenInfo = this.screenList[this.selectScreenIndex];

        udp4.send('connect', client.port, client.ip);

        // io.emit('requireConnect', {
        //   screenInfo: screenInfo,
        //   socketId: client.socketId,
        // });
      },
      refresh() {
        this.clientList.splice(0, this.clientList.length);
        udp4.send('server-online', 3001, '192.168.31.255');
      },
      startUdp() {

        udp4.on('error', (err) => {
          console.log(`服务器异常：\n${err.stack}`);
          udp4.close();
        });
        udp4.on('message', (msg, rinfo) => {
          let msg_str = msg.toString();
          console.log(`服务器收到：${msg_str} 来自 ${rinfo.address}:${rinfo.port}`);
          let client_ip = rinfo.address;
          let client_port = rinfo.port;
          if (/client-info===/.test(msg_str)) {
            let client_name = msg_str.replace(/client-info===/, '');

            let client = this.clientList.find(item => item.ip === client_ip);

            if (client) {
              client.port = client_port;
              client.name = client_name;
            } else {
              this.clientList.push({
                ip: client_ip,
                port: client_port,
                name: client_name,
              })
            }
          } else if ("client-offline" === msg_str) {
            let index = this.clientList.findIndex(item => item.ip === client_ip);
            if (index > -1) this.clientList.splice(index, 1);
          }
        });
        udp4.on('listening', () => {
          const address = udp4.address();
          console.log(`服务器监听 ${address.address}:${address.port}`);

          //服务端上线了
          // 向该网段所有ip发送上线通知
          console.log("发送上线通知")
          udp4.send('server-online', 3001, '192.168.31.255');
        });
        udp4.bind(3000);


      },
      window_close() {
        this.messageBox("是否退出？", "提示", {showCancelButton: true}).then(() => {
          remote.getCurrentWindow().close();
        });
      },
      window_min() {
        remote.getCurrentWindow().minimize();
      },
      dialogConfirm() {
        this.eventBus.emit("dialogConfirm");
      },
      dialogCancel() {
        this.eventBus.emit("dialogCancel");
      },
    },
  }
</script>

<style scoped lang="scss">
  #app {
    border-radius: 5px;
    box-sizing: border-box;
    overflow: hidden;
    background-color: #397eff;
    width: 100vw;
    height: 100vh;
    -webkit-app-region: drag;
    -webkit-user-select: none;
    .app_title {
      height: 40px;
      position: relative;
      .app_name {
        font-size: 20px;
        color: white;
        font-weight: bold;
        line-height: 40px;
        text-indent: 20px;
      }
      .app_control {
        position: absolute;
        top: 0;
        right: 0;
        height: 40px;
        font-size: 0;
        .app_control_btn {
          display: inline-block;
          width: 40px;
          height: 40px;
          img {
            -webkit-app-region: no-drag;
            box-sizing: border-box;
            width: 100%;
            height: 100%;
            padding: 13px;
            &:hover {
              background-color: rgba(204, 204, 204, 0.2);
            }
          }
        }
      }
    }
    $item_count: 4;
    $item_height: 40px;
    $item_space: 10px;
    .app_container {
      width: 80%;
      margin: 50px auto 0;
      font-size: 0;
      .list_panel {
        vertical-align: top;
        display: inline-block;
        width: 50%;
        box-sizing: border-box;
        padding: 10px;
        .title {
          border-bottom: 1px solid white;
          position: relative;
          .text {
            color: #fff;
            font-size: 20px;
            font-weight: bolder;
            box-shadow: 0 3px 0 white;
          }
          .refresh {
            position: absolute;
            top: 0;
            right: 0;
            font-size: 14px;
            color: #fff;
            cursor: pointer;
            -webkit-app-region: no-drag;
            padding: 2px 5px;
            border-radius: 3px;
            &:hover {
              background-color: rgba(204, 204, 204, 0.5);
            }
          }
        }
        .list {
          -webkit-app-region: no-drag;
          height: calc(#{$item_height} * #{$item_count} + #{$item_space} * #{$item_count});
          overflow-y: auto;
          .item {
            height: $item_height;
            line-height: $item_height;
            border-radius: 5px;
            margin-top: $item_space;
            text-indent: 10px;
            .text {
              font-size: 18px;
              color: white;
              cursor: default;
            }

            &.selected {
              background-color: #2755ac;
            }
          }
        }
      }
    }
    .btn_box {
      margin-top: 50px;
      text-align: center;
      button {
        outline: none;
        -webkit-app-region: no-drag;
        height: 40px;
        width: 150px;
        border-radius: 20px;
        background-color: white;
        border: none;
        font-size: 16px;
        color: #3a7fff;
        font-weight: bold;
        &:hover {
          background-color: antiquewhite;
          cursor: pointer;
        }
      }
    }

    .dialog {
      -webkit-app-region: no-drag;
      position: fixed;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      background-color: rgba(0, 0, 0, 0.5);
      .dialog_container {
        width: 400px;
        background-color: white;
        border-radius: 10px;
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        .title {
          font-size: 30px;
          text-indent: 30px;
          margin-top: 20px;
          height: 40px;
        }
        .msg {
          margin-top: 20px;
          padding: 0 30px;
        }
        .buttons {
          text-align: right;
          padding-right: 30px;
          margin-bottom: 20px;
          button {
            height: 30px;
            width: 80px;
            background-color: white;
            border: 1px solid #ccc;
            outline: none;
          }
        }
      }
    }
  }
</style>
