# Electron 对接讯飞语音唤醒WindowsSDK

## 一、项目主要依赖

 - vue
 - vue-cli-plugin-electron-builder
 - electron
 - ffi-napi nodejs操作c++的dll库
 - ref-napi c++类型转换
 - js-audio-recorder 录音插件

## 二、下载SDK
设置好唤醒词后, 下载windowsSdk, 项目需要`/bin`目录下的`msc_x64.dll` 和 `msc.dll` (分别是64位和32位的dll, 按需使用), 以及`/bin/msc/res/ivw`目录下的`wakeupresource.jet`(语音唤醒资源文件)

## 三、配置项目
### 1. 配置externals, 用于调用第三方库
```js
module.exports = {
  pluginOptions: {
    electronBuilder: {
      externals: ['ffi-napi', 'ref-napi'],
    }
  }
}
```

### 2. 配置sdk路径
由于开发模式和打包后的环境, 文件路径会产生差别, 所以需要将打包后的sdk路径进行配置

例如将sdk放在根目录的`libs`文件夹下, 则可以按下面配置:
```js
module.exports = {
  pluginOptions: {
    electronBuilder: {
      builderOptions: {
        extraResources: {
          from: 'libs/',
          to: './'
        }
      }
    }
  }
}
```

在代码中配置路径时, 需要进行判断
```js
let libPath = path.resolve('libs/bin/msc_x64.dll')
if (process.env.NODE_ENV !== 'development') {
  libPath = path.resolve('resources/bin/msc_x64.dll')
}
```

### 3. 通过ffi调用dll

![调用流程](https://www.xfyun.cn/doc/old_imges/msc_windows_images/QIVW.png)

#### 1. 主要需要使用sdk的以下方法:

 - MSPLogin 登录方法
 - QIVWSessionBegin 开启语音唤醒
 - QIVWRegisterNotify 注册唤醒监听事件
 - QIVWAudioWrite 写入音频


头文件都可以在下载的sdk的`include`文件夹找到
```c++
int MSPAPI MSPLogin(const char* usr, const char* pwd, const char* params);

typedef int( *ivw_ntf_handler)( const char *sessionID, int msg, int param1, int param2, const void *info, void *userData );

const char* MSPAPI QIVWSessionBegin(const char *grammarList, const char *params, int *errorCode);

int MSPAPI QIVWSessionEnd(const char *sessionID, const char *hints);

int MSPAPI QIVWAudioWrite(const char *sessionID, const void *audioData, unsigned int audioLen, int audioStatus);

int MSPAPI QIVWRegisterNotify(const char *sessionID, ivw_ntf_handler msgProcCb, void *userData);

int MSPAPI QIVWGetResInfo(const char *resPath, char *resInfo, unsigned int *infoLen, const char *params);

```

#### 2. ffi配置方法定义
方法的类型需要用到`ref-napi`进行转义
例如,
```
char* => string
int => ref.types.int
int* => ref.refType(ref.types.int)
unsigned int => ref.types.uint
回调方法 => 'pointer'
```

**注意:** `char*` 和 `char *`性质是一样的, 都是字符串

所以, 可以把需要的方法定义如下,
```js
const libm = ffi.Library(libPath, {
  MSPLogin: [ref.types.int, ['string', 'string', 'string']],
  QIVWSessionBegin: ['string', ['string', 'string', ref.refType(ref.types.int)]],
  QIVWSessionEnd: [ref.types.int, ['string', 'string']],
  QIVWAudioWrite: [ref.types.int, ['string', ref.refType(ref.types.void), ref.types.uint, ref.types.int]],
  QIVWRegisterNotify: [ref.types.int, ['string', 'pointer', ref.refType(ref.types.void)]]
})
```

方法的使用,就是通过 `libm.MSPLogin()`来调用即可

唯一需要注意的就是`QIVWRegisterNotify`方法, 需要传入的是一个回调函数, 上面定义时, 可以使用'pointer'占位

在调用时, 需要使用`ffi.Callback`创建一个回调, 传入函数中, 例如:

先查看回调函数的定义
```c++
typedef int( *ivw_ntf_handler)( const char *sessionID, int msg, int param1, int param2, const void *info, void *userData );

```

`ffi.Callback`的第一个参数是返回参数的类型, 第二个参数是传入回调函数的参数类型, 第三个参数是回调的处理

```js
const notifyCallback = ffi.Callback(
  ref.types.int,
  ['string', ref.types.int, ref.types.int, ref.types.int, ref.refType(ref.types.void), ref.refType(ref.types.void)],
  (sessionID, msg, param1, param2, info, userData) => {
    if (msg === 2) {
      console.log('err', param1)
    } else if (msg === 1) {
      console.log('唤醒了', info, userData)
      this.wakeText = '唤醒了' + Date.now()
    }
    global.sharedObj = notifyCallback
    return 0
  }
)

const notifyResCode = libm.QIVWRegisterNotify(this.sessionId, notifyCallback, null)

```

注意, 由于callback会被垃圾回收, 所以需要在调用的时候, 赋值到一个全局变量上, 比如`global['变量名'] = 回调函数`


### 3. 实时录音传递
初始化录音, 使用单声道, 16位, 16000采样率
```js
this.recorder = new Recorder({
  sampleBits: 16, // 采样位数，支持 8 或 16，默认是16
  sampleRate: 16000, // 采样率，支持 11025、16000、22050、24000、44100、48000，根据浏览器默认值，我的chrome是48000
  numChannels: 1,
  compiling: true
})
```

需要使用`js-audio-recorder`的 V0.5.7 版本, 通过定时调用`getNextData`方法, 获取当前音频转成`buffer`传入`QIVWAudioWrite`方法

由于录音是一直存在缓存中的, 时间长了就会把内存占满, 导致程序崩了.

而我们使用语音唤醒, 不需要留存录音, 所以需要对使用过的音频缓存进行清除

当前的库里清除缓存的方法是`clear`, 而`clear`方法没有清除`tempPCM`, 还是会导致问题, 所以需要重新写个方法, 重新打包

```js
clearCache(): void {
  this.lBuffer.length = 0;
  this.rBuffer.length = 0;
  this.size = 0;
  this.fileSize = 0;
  this.PCM = null;
  this.tempPCM = []
  this.audioInput = null;
  this.duration = 0;
  this.ispause = false;
  this.isplaying = false;
  this.playTime = 0;
  this.totalPlayTime = 0;
}

```

之后, 我们就可以定时调用下面方法, 来进行音频写入了

```js
async getBuffer () {
  const data = this.recorder.getWholeData()
  let arr = []
  data.forEach(d => {
    arr = arr.concat(...Buffer.from(d.buffer))
  })
  if (this.isBegin) {
    const buffer = Buffer.from(arr)
    if (buffer.length === 0) {
      return
    }
    // this.ws.send(buffer)
    const writeRes = libm.QIVWAudioWrite(this.sessionId, buffer, buffer.length, 2)
    if (writeRes !== 0) {
      console.log('写入失败')
    }
  }
  this.recorder.clearCache()
}
```

### 4. 参考资料
1. https://www.xfyun.cn/doc/asr/awaken/Windows-SDK.html#_2%E3%80%81sdk%E9%9B%86%E6%88%90%E6%8C%87%E5%8D%97
2. https://juejin.cn/post/6844903645905977357