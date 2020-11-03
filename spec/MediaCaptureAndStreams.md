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