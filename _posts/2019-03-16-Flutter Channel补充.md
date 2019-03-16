---
layout: post
title: Flutter Channel补充
feature-img: "assets/img/background/pexels-photo5.jpeg"
tags: [Flutter,Android]
---

> 共你促膝把酒，倾通宵都不够。 <br>                
> <p align="right">——《最佳损友》</p>


## 继承关系

在Channel中，Messenger负责消息的发送和接受，下面是实现关系

![](https://i.loli.net/2019/03/16/5c8cb11b320f0.jpg)

其中这个FlutterView是我们在嵌入Flutter的时候用到的，很有意思的是，继承了SufaceView。

{% highlight java%}

        FlutterView flutterView = Flutter.createView(this, getLifecycle(), "route1");
        FrameLayout.LayoutParams layoutParams = new FrameLayout.LayoutParams(FrameLayout.LayoutParams.MATCH_PARENT, FrameLayout.LayoutParams.MATCH_PARENT);
        frameLayout.addView(flutterView, layoutParams);

{% endhighlight %}


## 一点对比


### 三种Channel对比

Flutter一共提供三种Channel

* BasicMessageChannel: 支持字符串和半结构化的数据。

* MethodChannel: 传递方法调用。Flutter主动调用Native的方法，并获取相应的返回值。比如官方示例给的，获取系统电量，发起Toast等调用系统API，可以通过这个来完成。

* EventChannel: 传递事件，支持数据流通信。这里是Native将事件通知到Flutter。比如Flutter需要监听网络情况，这时候MethodChannel就无法胜任这个需求了。EventChannel可以将Flutter的一个监听交给Native，Native去做网络广播的监听，当收到广播后借助EventChannel调用Flutter注册的监听，完成对Flutter的事件通知。


### 三种Handler对比

每种Channel里面都有一个对应的Handler

* BasicMessageChannel：MessageHandler

{% highlight java%}

    public interface MessageHandler<T> {
        void onMessage(T var1, BasicMessageChannel.Reply<T> var2);
    }

{% endhighlight %}

`onMessage`参数为泛型的消息和一个Reply用于返回消息


* MethodChannel：MethodCallHandler

{% highlight java%}

    public interface MethodCallHandler {
        void onMethodCall(MethodCall var1, MethodChannel.Result var2);
    }

{% endhighlight %}

`onMethodCall`参数为一个`MethodCall`对象和一个Reslut用于返回消息

* EventChannel：StreamHandler

{% highlight java%}

    public interface StreamHandler {
        void onListen(Object var1, EventChannel.EventSink var2);
        void onCancel(Object var1);
    }

{% endhighlight %}

当Flutter端开始监听平台事件时,会向平台发起一次MethodCall,其中方法名为listen,也就是最终会调用StreamHandler中的onListen()方法.该方法中接受两个参数,其中EventSink类型的参数可用于向Flutter发送事件(实际上还是通过BinaryMessager).当Flutter开始停止监听平台事件时,会再向平台发起一次MethodCall,其中方法名为cancel,最终会调用StreamHandler的onCancel()方法,在该方法中通常需要销毁一些无用的资源。

见下面

{% highlight java%}

Stream<dynamic> receiveBroadcastStream([dynamic arguments]) {
    final MethodChannel methodChannel = MethodChannel(name, codec);
    StreamController<dynamic> controller;
    controller = StreamController<dynamic>.broadcast(onListen: () async {
      BinaryMessages.setMessageHandler(name, (ByteData reply) async {
      /...../
      try {
        await methodChannel.invokeMethod<void>('listen', arguments);
      } catch (exception, stack) {
        FlutterError.reportError(FlutterErrorDetails(
        /...../
      }
    }, onCancel: () async {
      BinaryMessages.setMessageHandler(name, null);
      try {
        await methodChannel.invokeMethod<void>('cancel', arguments);
      } catch (exception, stack) {
        /...../
        ));
      }
    });
    return controller.stream;
  }

{% endhighlight %}
 

### 与ReactNative对比

### Dart异步

### 一点测试

测试机型：华为 SCL-CL00 Android 5.1.1 （性能很低端的手机）

使用MethodChannel。

#### List<String>传输速度测试

数据：7000*1000个汉字
耗时：平均3.15s左右

![](https://i.loli.net/2019/03/16/5c8c97d93c78f.jpg)



#### Map<int,String>传输速度测试

数据：7000*1000字符长度
耗时：平均3.4s左右

![](https://i.loli.net/2019/03/16/5c8c97d93a8f4.jpg)



#### JSON数据 HashMap

{
      "message": "登录成功",
      "success": true,
      "data": {
        "id": "2c9a821c67722c8f01678bce27090064",
        "nickName": "123",
        "portrait": "http://ecommunity.oss-cn-zhangjiakou.aliyuncs.com/1544523304800.jpg",
        "phone": "15662383508",
        "reviewStatus": false,
        "houseResult": null,
        "communityResult": null,
        "tags": [
          "0"
        ],
        "token": "536D901709D28C29652D0100A6D97635",
        "marketSkillCertification": false
      }
}

![](https://i.loli.net/2019/03/16/5c8c97d928e95.jpg)
