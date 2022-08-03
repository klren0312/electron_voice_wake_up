<template>
  <div class="main-page">
    <div class="debug-block">
      <div class="recorder-data" v-if="isRecording">
        <span>正在录音</span>
      </div>
      <button @click="startAudioFile(1)">文件唤醒(成功)</button>
      <button @click="startAudioFile(0)">文件唤醒(失败)</button>
      <h1>{{ wakeText }}</h1>
    </div>
  </div>
</template>
<script>
import fs from 'fs'
import ffi from 'ffi-napi'
import ref from 'ref-napi'
import path from 'path'
import Recorder from '@/assets/js/recorder'

let libPath = path.resolve('libs/bin/msc_x64.dll')
if (process.env.NODE_ENV !== 'development') {
  libPath = path.resolve('resources/bin/msc_x64.dll')
}
const libm = ffi.Library(libPath, {
  MSPLogin: [ref.types.int, ['string', 'string', 'string']],
  MSPLogout: [ref.types.int, []],
  QIVWSessionBegin: [
    'string',
    ['string', 'string', ref.refType(ref.types.int)],
  ],
  QIVWSessionEnd: [ref.types.int, ['string', 'string']],
  QIVWAudioWrite: [
    ref.types.int,
    ['string', ref.refType(ref.types.void), ref.types.uint, ref.types.int],
  ],
  QIVWRegisterNotify: [
    ref.types.int,
    ['string', 'pointer', ref.refType(ref.types.void)],
  ],
})
// 给识别用, 防止回调被GC
// https://blog.csdn.net/weixin_44148064/article/details/103514292
global.sharedObj = {}
export default {
  name: 'App',
  data() {
    return {
      showPeople: false,
      recorder: null,
      recorderData: {
        vol: 0,
        duration: 0,
      },
      sessionId: '',
      isRecording: false,
      isLogin: false,
      isBegin: false,
      wakeText: '',
    }
  },
  created() {
    this.initRecorder()
    console.log(`isLogin: ${this.isLogin}, isBegin: ${this.isBegin}`)
    // 登录sdk
    if (this.isLogin === false) {
      const loginStr = `appid = ${process.env.VUE_APP_APPID}, engine_start = ivw, work_dir = .`
      const loginRes = libm.MSPLogin(null, null, loginStr)
      if (loginRes === 0) {
        console.log('登录成功')
        this.isLogin = true
      } else {
        // 登录错误
        console.error(loginRes)
      }
    }

    if (this.isLogin && this.isBegin === false) {
      let resPath = path.resolve('libs/bin/msc/res/ivw/wakeupresource.jet')
      if (process.env.NODE_ENV !== 'development') {
        resPath = path.resolve('resources/bin/msc/res/ivw/wakeupresource.jet')
      }
      const ssbStr = `ivw_threshold=0:1450,sst=wakeup,ivw_res_path =fo|${resPath}`
      const beginResCode = ref.alloc(ref.types.int, 0)
      // 开启识别
      this.sessionId = libm.QIVWSessionBegin(null, ssbStr, beginResCode)
      if (beginResCode.deref() === 0) {
        this.isBegin = true
        console.log('开启识别成功')
        // 开启监听
        const notifyCallback = ffi.Callback(
          ref.types.int,
          [
            'string',
            ref.types.int,
            ref.types.int,
            ref.types.int,
            ref.refType(ref.types.void),
            ref.refType(ref.types.void),
          ],
          (sessionID, msg, param1, param2, info, userData) => {
            if (msg === 2) {
              console.log('err', param1)
            } else if (msg === 1) {
              console.log('唤醒了', info, userData)
              this.wakeText = '唤醒了' + Date.now()
              this.showPeople = true
              setTimeout(() => {
                this.showPeople = false
              }, 4000)
            }
            global.sharedObj = notifyCallback
            return 0
          }
        )

        const notifyResCode = libm.QIVWRegisterNotify(
          this.sessionId,
          notifyCallback,
          null
        )
        console.log('notifyResCode', notifyResCode)
      }
    }

    setInterval(() => {
      this.getBuffer()
    }, 200)
  },
  beforeDestroy() {
    console.log('beforeDestroy')
    this.isLogin && this.doQIVWLogout()
    this.isBegin && this.stopQIVW()
  },
  methods: {
    initRecorder() {
      this.recorder = new Recorder({
        sampleBits: 16, // 采样位数，支持 8 或 16，默认是16
        sampleRate: 16000, // 采样率，支持 11025、16000、22050、24000、44100、48000，根据浏览器默认值，我的chrome是48000
        numChannels: 1,
        compiling: true,
      })
      this.recorder.onprogress = (params) => {
        this.isRecording = true
        // console.log(params.data) // 当前获取到到音频数据
        this.recorderData = {
          duration: params.duration.toFixed(2),
          vol: params.vol.toFixed(2),
        }
      }

      this.recorder.start().then(
        () => {
          console.log('开始录音')
        },
        (error) => {
          console.log(`异常了,${error.name}:${error.message}`)
        }
      )
    },

    async getBuffer() {
      const data = this.recorder.getWholeData()
      let arr = []
      data.forEach((d) => {
        arr = arr.concat(...Buffer.from(d.buffer))
      })
      if (this.isBegin) {
        const buffer = Buffer.from(arr)
        if (buffer.length === 0) {
          return
        }
        const writeRes = libm.QIVWAudioWrite(
          this.sessionId,
          buffer,
          buffer.length,
          2
        )
        if (writeRes !== 0) {
          console.log('写入失败')
        }
      }
      this.recorder.clearCache()
    },
    /**
     * 音频文件识别测试
     * @param {number} type 1-成功唤醒的音频 0-唤醒失败的音频
     */
    startAudioFile(type) {
      if (this.isBegin) {
        // 音频输入
        let audioPath = path.resolve(
          `libs/bin/audio/${type === 1 ? 'success' : 'error'}.wav`
        )
        if (process.env.NODE_ENV !== 'development') {
          audioPath = path.resolve(
            `resources/bin/audio/${type === 1 ? 'success' : 'error'}.wav`
          )
        }
        const audioBuffer = fs.readFileSync(audioPath)
        const size = audioBuffer.length
        const writeRes = libm.QIVWAudioWrite(
          this.sessionId,
          audioBuffer,
          size,
          2
        )
        console.log('音频写入', writeRes)
      }
    },
    stopQIVW() {
      if (this.isBegin) {
        const resCode = libm.QIVWSessionEnd(this.sessionId, '页面关闭')
        if (resCode === 0) {
          this.isBegin = false
          console.log('关闭识别成功')
        }
      }
    },
    doQIVWLogout() {
      if (this.isLogin) {
        const resCode = libm.MSPLogout()
        if (resCode === 0) {
          this.isLogin = false
          console.log('登出')
        }
      }
    },
  },
}
</script>
<style>
html,
body {
  margin: 0;
  padding: 0;
}
.recorder-data {
  text-align: left;
  font-size: 12px;
}
.main-page {
  position: relative;
  width: 100%;
  height: 100vh;
  text-align: center;
}
</style>
