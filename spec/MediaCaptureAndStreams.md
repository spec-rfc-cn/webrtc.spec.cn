# 媒体捕获和流

w3c[地址](https://www.w3.org/TR/mediacapture-streams/#dom-mediastream),
翻译版本: 20201007.

编辑者有4位:

- 思科
- 微软
- 火狐
- 谷歌

## 摘要

本spec提供了平台级的js api,也就是说提供了平台无关的api,
通过这些api可以访问本地媒体(音视频).

## 介绍

ps:本章是`非标准的`.

本spec提供的api用于请求访问本地多媒体设备(麦克风/摄像头).
同时还定义了MediaStream api(这类api可以控制媒体数据被谁消费,
以及设备产生媒体数据时的部分控制).
除此之外,还暴露了设备被捕获和媒体渲染的部分信息.

## 一致性

同webrtc spec

## 术语

### html中的EventHandler接口

同webrtc spec,这个接口表示事件处理中的一个回调.

### html中的queue a task和 fire an event

同webrtc spec

### html中的current settings object

表示环境设置对象,再深入就到了ecmascript内的概念了.

### hmtl中的allowed to used

允许使用,字面意思

### html中的fulfilled,rejected,resolved

其实定义在ecma-262中,太复杂,直接百度的.

js的Promises用于异步处理,有3种状态:未完成,完成,失败.
fulfilled就表示完成,在构造promise时,可指定两个回调,
分别是resolve和reject,当异步任务执行完,调用resolve;
反之执行失败就调用reject回调.

    const myFirstPromise = new Promise((resolve, reject) => {
      // ?做一些异步操作，最终会调用下面两者之一:
      //
      //   resolve(someValue); // fulfilled
      // ?或
      //   reject("failure reason"); // rejected
    });
    
这块涉及到js的异步编程,后面遇到了再详细补充.

### dom中的Event,EventInit,EventTarget

Event是接口,EventInit是结构体,Event构造时参数包含EventInit.
其他同webrtc spec.

### DOMException

webidl中定义的,是个接口,封装了一些常见的异常信息.

### source

提供媒体流轨道源的某个东西,叫source.也可理解为广播媒体的广播者.
source可以是物理摄像头/麦克风,磁盘上的本地音视频文件,
也可以是网络资源,静态图片等.只要是提供媒体的都可以称为source.
__本spec只描述摄像头/麦克风类型的源,其他类型的源在其他标准里描述__.

如果程序未经授权,去访问source时,只能获取source的数量/类型/和其他设备的关系.
只有授权之后,才可以通过source的附加信息来使用source.
后面的访问控制模型会重点描述这块.

source没有约束,但track有.当source和track连接时,
source产生的媒体就要符合track上的约束.多个track都已附加到一个source上.
user agent的处理(eg:下采样)用于保证所有的track都有与其相符的媒体.

source的约束属性,其实是由track暴露的,暴露的有两种:
capabilities和settings.只是看起来约束属于source,
这导致source可以满足不同的需求.capabilities针对一个source的任意track是通用的,
settings对每个track就略有区别.
eg:一个source有两个track,对track查询capability和setting信息,
获取的capability是相同的,而setting是根据约束的不同而有所不同.

### source里的setting

setting表示source的约束属性,只读.

source的环境可能是动态改变的(eg:摄像头的帧率变化).
这种情况下,受影响的track可能不再满足约束.
整个api层应该尽量减少这类影响,另外,即使不再满足约束,
也应该继续传送媒体.

setting虽然属于source的属性,但只通过source的track暴露给应用程序.
暴露的接口是ConstrainablePattern.

### source里的capability

对于每个可约束的属性,会有一个capability来表示source是否支持这个约束属性,
以及约束属性的取值范围.
capability通过ConstrainablePattern接口暴露.

支持capabilities的值,在spec回规定取值范围,或枚举范围.

对于一个track调用getCapabilities()函数,会返回source的capabilities,
也就是说,会返回source中所有track的的能力(capabilities,后面翻译为能力).

source的能力是一个常量.针对一个source,应用程序不管在哪个浏览会话中,
获取到的source能力都是一致的.

关于能力的设计就是这么简单.能力不是用于描述不同值之间的关系.
eg:摄像头提供视频流可以是高分辨率低帧率,也可以是低分辨率高帧率的,
能力不是用于描述这两种的区别的.能力用于描述值的范围.
通过试图应用约束来暴露约束之间的关系.

ps:专业术语就是难懂.我记得取某款摄像头支持的能力时,获取的如下:
(rgb-30fps-720p),(rgb-10fps-1080p),(rgb-5fps-1080p),
(yuv-30fps-1080p),(rgb-30fps-720p),(rgb-30fps-360p),
可以看到里面有3个约束值,颜色空间/帧率/分辨率,映射到上面的capability,
能力是颜色空间支持rgb/yuv,帧率支持5-30fps,分辨率支持360-1080p,
当视图用一些约束来调用摄像头时(eg:1080p),此时会暴露下列约束间的关系:
(rgb-10fps-1080p),(rgb-5fps-1080p),(yuv-30fps-1080p).

### source里的constraints

constraints,约束条件,应用程序通过constraints可以控制两件事:
一是选择适当的source(其实是track),而是选中source后,可以影响source的操作.

约束条件(constraints后续翻译为约束条件)限制了source能用的操作模式.
如果没有约束条件,在实现时,source的settings就是从source支持的能力中选取.
实现也可以在任何时候调整souce的settings,当然要在约束支持的范围内调整.

getUserMedia()就可以利用约束条件来选择合适的source并进行配置.
在使用getUserMedia()之后,还可以通过ConstrainablePattern接口动态调整
track的约束.

如果在调用getUserMedia()函数时,一旦初始约束没有满足,track不会连接到source.
不管怎样,track满足约束这点,可能会改变,一旦约束不再满足,
需要通过ConstrainablePattern接口来通知应用程序.

对于每个约束属性,会有一个相应的setting名称和capability名称.

约束分3组:

- required constraints, 非高级约束都是必需约束
- optional basic constraints, 非高级约束,可选基础约束
- advanced constraints, 高级约束,用advanced关键字描述

一般来说,应用的约束条件越少,user agents就能更灵活地对媒体流进行优化,
因此,应用程序的开发者,应该尽量少用required constraints,或者谨慎地用.

### RTCPeerConnection

webrtc spec中定义的

### Permissions

permissions标准中定义.

reading current permission state, 获取当前用户权限,
request permission to use are, 询问用户.

## MediaStream API

### 介绍

MediaStream API分两部分:MediaStreamTrack/MediaStream接口.
MediaStreamTrack对象表示user agent中来自一个媒体source的单个类型媒体.
MediaStream表示一组MediaStreamTrack对象,
MediaStream可以在media element(eg:网页的video/audio元素)中录制或渲染.

一个MediaStream可以包含0个或多个MediaStreamTrack,在渲染时,
内部track需要保持同步.这点不是硬指标,因为track的时钟可能会不同.
不同的MediaStream没必要保持同步.

ps:track同步固然好,但某些场合牺牲同步也是值得的.
特别是track是远端source或是实时流(eg:webrtc流),牺牲同步比延时累积或其他风险
好太多了.对于实现来说,track同步最好,但在某些场景需要在同步和用户感知做取舍.

一个MediaStreamTrack可以表示多通道内容,eg:立体音,立体视频(3D视频).
通道channel的api是通过其他协议暴露的,eg:webaudio协议,不过这个不提供直接访问通道.

MediaStream对象包含一个输入和一个输出,组合了所有track的输入和输出.
输出决定了渲染(html中video的播放或录制成文件).一个MediaStream可同时
附加到不同的输出中.

一个新的MediaStream对象,可以从现有媒体stream/track中创建,
做法是调用MediaStream()构造函数,构造函数里可以带参数,
参数可以是已存在的MediaStream对象,或一组MediaStreamTrack对象.

MediaStream/MediaStreamTrack对象都可以cloned,克隆对象的track和被克隆的对象一致.
只是track的约束不一样,这样设计是方便适配不同的消费者.
MeidaStream也在getUserMedia以外的场景中用到,eg:webrtc.

### MediaStream

构造函数是在现有track上构造一个新的stream(后面用stream指MediaStream对象).
如上面所讲,可带两个可选参数.

    [Exposed=Window]
    interface MediaStream : EventTarget {
      constructor();
      constructor(MediaStream stream);
      constructor(sequence<MediaStreamTrack> tracks);
      readonly attribute DOMString id;
      sequence<MediaStreamTrack> getAudioTracks();
      sequence<MediaStreamTrack> getVideoTracks();
      sequence<MediaStreamTrack> getTracks();
      MediaStreamTrack? getTrackById(DOMString trackId);
      undefined addTrack(MediaStreamTrack track);
      undefined removeTrack(MediaStreamTrack track);
      MediaStream clone();
      readonly attribute boolean active;
      attribute EventHandler onaddtrack;
      attribute EventHandler onremovetrack;
    };

3个构造函数,一个无参,一个带MediaStream,一个带一组MeidaStreamTrack.

构造函数的步骤如下:

- 构造一个新的MediaStream对象叫stream
- stream.id = 新值
- 如果有参数,走以下流程
  - 新建变量 tracks = 空的MediaStreamTrack集合
    - 如果参数是MediaStream对象
      - tracks = MediaStream对象的所有MediaStreamtrack
    - 如果参数是一组MediaStreamTrack对象
      - tracks = 这组MediaStreamTrack
  - 遍历tracks
    - 如果track已经在stream的轨道集合中,跳过
    - 否则,将track添加到stream的轨道集合
- 返回stream

MediaStream内部会有一个轨道集合.轨道集合中的顺序由user agent定义,
API对顺序是没有要求的.在轨道集合中查找,需要通过track.id实现.

从MediaStream的输出读取数据,称为MediaStream的消费者.目前有以下消费者:

- html中的媒体元素, video/audio
- webrtc的RTCPeerConnection
- MediaRecorder中的媒体录制
- ImageCapture中的照片捕获
- MediaStreamAudioSourceNode中的web aduio

每个消费者都需要支持track的添加和删除.

stream的active,表示至少还有一个track没有ended,
如果stream没有track或拥有的track都是ended,那么stream就是inactive.

stream的audible,表示至少有一个非ended的audio track.
如果stream没有任何音频track,或拥有的audio track都是ended,称为inaudible.

user agent可能会更新stream的轨道集合.本spec没有针对这种情况做描述,
但其他协议可能会.webrtc 1.0 就针对这点做了描述.

stream的 add a track:

- 如果track已经在stream的轨道集合中,退出
- 将track添加到stream的轨道集合
- 在stream中触发一个addtrack的事件,参数就是track

stream的 remove a track:

- 如果track不在stream的轨道集合中
- 从stream的轨道集合中移除track
- 在stream中触发一个removetrack的事件,参数就是track

MediaStream接口分析:
继承于EventTarget,事件目标接口,3个构造函数(前面已经分析了的),
id属性,标志stream的属性,3个获取track集合的函数,以及1个通过track id获取track集合,
对track的增删以及对增删事件的监听,对stream的clone,最后是active状态.

- id属性
  - MediaStream构造时,由user agent生成的标志字符串
  - 最好是用uuid,36个字符
- active属性
  - 表示stream是否是active状态
- onaddtrack/onremovetrack
  - 是事件addtrack/removetrack的事件处理
- getAudioTracks方法
  - 返回stream中"audio track分组"
  - 本spec并没有指定track集合到"track分组"的转换,这个由user agent定义
  - 本spec也没有指定两次调用要保持一致的顺序
- getVideoTracks方法/getTracks方法
  - 同getAudioTracks方法
- getTrackById方法
  - 根据track id查找,没找到就返回null
  - 找到就返回对应的MediaStreamTrack
- addTrack方法/removeTrack方法
  - 具体过程如上面的add a track/ remove a track
- clone方法
  - 克隆stream和所有的track,具体流程如下
  - streamClone = 新构造的MediaStream对象
  - streamClone.id = 新uuid值
  - 克隆每个track,并添加到streamClone的轨道集合
  - 返回streamClone

克隆每个track,下节谈到.

### MediaStreamTrack

在user agent上,MediaStreamTrack表示一个媒体source.
连接到user agent的源,最常见的就是设备.
注意,其他sepc对MediaStreamTrack的source有不同定义,或许会覆盖本
spec定义的默认行为.同一个媒体source可以用多个track来表示,
界面上选择同一摄像头,会导致调用两次getUserMeida,
此时媒体source(摄像头)没有变,却包含两个不同的track.

从MediaStreamTrack对象出来的数据,不一定要是规范的二进制格式.
这样是允许user agent来维护媒体.

如果source调用了stop()方法,那么MediaStreamTrack对象就不再需要了.
不管通过什么方式,只要source的所有track被停止(be stopped或be ended),
那么source就是stopped状态.如果source是通过getUserMeida()暴露的设备,
当source停止时(is stopped),user agents(有时简称UA)执行以下操作;

    deviceId = 设备的设备id
    devicesLiveMap[deviceId] = false
    获取设备类型和授权,如果不是允许,devicesAccessibleMap[deviceId] = false

实现可能会对source用引用计数来跟踪source的使用,本spec不谈这点.

clone a track, 上一节最后遗留的尾巴,UA会执行以下步骤:

- track = 要被克隆的MediaStreamTrack对象
- trackClone = 新构造的MediaStreamTrack对象
- trackClone.id = 新值
- 初始化trackClone的kind/label/readyState/enabled属性(从track复制)
- trackClone对应的source设置成track对应的source
- trackClone的约束(constraints)设置为track还active(活跃)的约束
- 返回trackClone

#### 生命周期和 media flow

先谈生命周期.

track的生命周期状态有两种:live和ended.在MediaStreamTrack构造时可指定状态,
两种状态均可指定.eg:克隆一个ended track,状态就是ended.
生命周期内的状态用MediaStreamTrack.readyState属性来表示.

如果是live状态,表示track是active的,生成的媒体可被消费者使用.
如果MediaStreamTrack是muted或disabled,内容会被替换成"zero-information-content".

当track是muted/disabled时,要么是发送静音/黑帧,要么是"zero-information-content",
两者是等价的.

如果source是通过getUserMedia()获取的,当track被muted/disabled时,
那和当前设备连接的所有track,会被 be muted/disabled/stopped.
此时UA可能会将`devicesLiveMap[设备id]`设置为fasle,
只要正在连接的track再次进行 un-muted/enabled,UA的设置会重新设回true.

当一个live,unmuted,enabled的track的source来至getUserMedia(),
当track被muted/disabled时,和当前设备连接的所有track,会被 be muted/disabled/stopped.
(这点是重复上面的描述),此时UA应该在3秒内放弃视图继续操作设备的,
这3秒是给用户来反应的.一旦某个连接的track再次unmuted/endabled时,
UA应该尽快获取设备,并聚焦于"当前设置对象"的"可响应document".
如果document没有获得焦点,UA应该在任务队列中添加一个mute任务,
直到document获取焦点后,UA再次在任务队列添加一个unmute的任务.
如果获取设备失败了,UA应该 end the track.(检测到设备问题,应该尽快结束相关track,
eg:设备的物理插拔).

end the track,稍后分析.

track的muted/unmuted状态反映了此时此刻source是否有提供媒体.
enabled/disabled状态是应用程序控制的,决定track是否输出媒体.
所以如果想让媒体数据继续按flow走,那么track对象要是unmuted和enabled.

当track是muted时,source暂时不能向track提供数据.
用户可以进行muted,通常这个操作不受应用程序控制.
UA也可以将一个track设置为muted.

应用程序可以设置track的enable/disable,决定是否渲染source的媒体.
不管track是不是enable,只要设置了muted,渲染就是黑的了.
以消费者的角度看,track的disabled和muted是一个意思.
但从source的角度看,两者是有区别的.

### MediaStreamTrackEvent