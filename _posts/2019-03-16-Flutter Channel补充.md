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
 

### 一点测试

测试机型：华为 SCL-CL00 Android 5.1.1 （性能低端手机）

#### MethodChannel List<String>传输速度测试

* 数据：4566*1500个汉字
* 耗时：平均3.242s

{% highlight java%}


    I/flutter (14132): 3.277
    I/flutter (14132): success
    I/flutter (14132): 3.185
    I/flutter (14132): success
    I/flutter (14132): 3.264
    I/flutter (14132): success
    I/flutter (14132): 3.231
    I/flutter (14132): success
    I/flutter (14132): 3.253
    I/flutter (14132): success


{% endhighlight %}


#### MethodChannel Map<int,String>传输速度测试

* 数据：4566*1500个汉字
* 耗时：平均3.4s左右(理所当然的慢)

![](https://i.loli.net/2019/03/16/5c8c97d93a8f4.jpg)

#### BasicMessageChannel BinaryCodec测试

* 数据：4566*1000个汉字
* 平均时间：2.241s

{% highlight java%}

    I/flutter (26424): 2.288s
    I/flutter (26424): 2.181
    I/flutter (26424): 2.253s

{% endhighlight %}


* 数据：4566*1500个汉字
* 平均时间：3.18s

{% highlight java%}

    I/flutter (12698): 3.164
    I/flutter (12698): 3.188
    I/flutter (12698): 3.193
    I/flutter (12698): 3.146
    I/flutter (12698): 3.22

{% endhighlight %}


* 数据：4566*2000个汉字
* 平均时间:4.188s


{% highlight java%}

    I/flutter (25770): 4.252s
    I/flutter (25770): 4.125s
    I/flutter (25770): 4.179s

{% endhighlight %}


#### BasicMessageChannel StandardMessageCodec测试

* 数据：4566*1000个汉字
* 平均时间：2.152
{% highlight java%}

    I/flutter (15318): 1000
    I/flutter (15318): 传送时间为：
    I/flutter (15318): 0.022
    I/flutter (15318): 总时间为：
    I/flutter (15318): 2.297
    I/flutter (15318): 1000
    I/flutter (15318): 传送时间为：
    I/flutter (15318): 0.012
    I/flutter (15318): 总时间为：
    I/flutter (15318): 2.105
    I/flutter (15318): 1000
    I/flutter (15318): 传送时间为：
    I/flutter (15318): 0.012
    I/flutter (15318): 总时间为：
    I/flutter (15318): 2.073

{% endhighlight %}



* 数据：4566*1500个汉字
* 平均时间：3.163s

{% highlight java%}


    I/flutter (15318): 1500
    I/flutter (15318): 传送时间为：
    I/flutter (15318): 0.019
    I/flutter (15318): 总时间为：
    I/flutter (15318): 3.163
    I/flutter (15318): 1500
    I/flutter (15318): 传送时间为：
    I/flutter (15318): 0.015
    I/flutter (15318): 总时间为：
    I/flutter (15318): 3.168
    I/flutter (15318): 1500
    I/flutter (15318): 传送时间为：
    I/flutter (15318): 0.013
    I/flutter (15318): 总时间为：
    I/flutter (15318): 3.159

{% endhighlight %}

* 数据：4566*2000个汉字
* 平均时间：4.104s
    
{% highlight java%}

    I/flutter (15318): 2000
    I/flutter (15318): 传送时间为：
    I/flutter (15318): 0.016
    I/flutter (15318): 总时间为：
    I/flutter (15318): 4.086
    I/flutter (15318): 2000
    I/flutter (15318): 传送时间为：
    I/flutter (15318): 0.019
    I/flutter (15318): 总时间为：
    I/flutter (15318): 4.138
    I/flutter (15318): 2000
    I/flutter (15318): 传送时间为：
    I/flutter (15318): 0.016
    I/flutter (15318): 总时间为：
    I/flutter (15318): 4.088

{% endhighlight %}





#### BasicMessageChannel 两种解码器 内存使用对比

数据：数据：4566*2500个汉字

* BinnaryCodec:

![](https://i.loli.net/2019/03/16/5c8cf11055334.jpg)


* StandardMessageCodec:


![](https://i.loli.net/2019/03/16/5c8cf110538da.jpg)



#### Native和Flutter编解码时间测试

* 数据：4566*1000汉字

* Native平均编码时间：1.195s
* Native平均解码时间：1.383s
* 合计：2.578s

* Flutter平均编码时间：0.832s
* Flutter平均解码时间：2.419
* 合计：3.521s

{% highlight java%}

    I/flutter ( 6371): Flutter编码时间
    I/flutter ( 6371): 0.92s
    I/flutter ( 6371): Flutter解码时间
    I/flutter ( 6371): 2.533s
    Native编码时间为1.389s
    Native解码时间为1.488s

    I/flutter ( 6371): Flutter编码时间
    I/flutter ( 6371): 0.772s
    I/flutter ( 6371): Flutter解码时间
    I/flutter ( 6371): 2.289s
    Native编码时间为1.13s
    Native解码时间为1.246s

    I/flutter ( 7856): Flutter编码时间
    I/flutter ( 7856): 0.863s
    I/flutter ( 7856): Flutter解码时间
    I/flutter ( 7856): 2.581s
    Native编码时间为1.178s
    Native解码时间为1.547s

    I/flutter ( 7856): Flutter编码时间
    I/flutter ( 7856): 0.774
    I/flutter ( 7856): Flutter解码时间
    I/flutter ( 7856): 2.273
    Native编码时间为1.086
    Native解码时间为1.252

{% endhighlight %}

### 一点结论

1. `BasicMessageChannel`与`BinaryCodec`结合更适合传递大数据块
2. 如果条件允许，尽量还是用byte数组吧，这样很大程度上减少编码和解码时间
3. Flutter的解码速度很快，但是解码速度很慢，Native编码和解码速度较为平均，结合两者来看还是Native要快









