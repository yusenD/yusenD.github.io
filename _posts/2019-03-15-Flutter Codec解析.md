---
layout: post
title: Flutter Codec解析
feature-img: "assets/img/background/pexels-photo18.jpeg"
tags: [Flutter,Android]
---

> 奋不顾身。 <br>

<br>

在之前的文章中可以看到，在Fluter提供的三种Channel中，都有下面的三个成员变量：

* name:表示Channel名字,每个Channel使用唯一的name作为其唯一标志
* messager:信使,是消息的发送和接受工具
* codec: 表示消息的编解码器,目前有MethodCodec和MessageCodec两种类型

Flutter实现了一套二进制传输“协议”，其中Codec功不可没，也是比较好奇。今天就结合源码看一下Codec的具体实现。

## 概述

在Flutter中，有两种Codec

* MethodCodec: 用于对MethodCall编解码
* MessageCodec: 用于对Message进行编解码

### MessageCodec

MessageCodec是一个泛型接口，用于二进制数据ByteBuffer与基础数据之间的编解码，并且提供了两个方法：

{% highlight java%}

    ByteBuffer encodeMessage(T var1);

    T decodeMessage(ByteBuffer var1);
    
{% endhighlight %}

它的具体实现类如下：

![](https://i.loli.net/2019/03/16/5c8cb0f07c1a8.jpg)

#### BinnaryCodec

{% highlight java%}
public final class BinaryCodec implements MessageCodec<ByteBuffer> {
    public static final BinaryCodec INSTANCE = new BinaryCodec();

    private BinaryCodec() {
    }

    public ByteBuffer encodeMessage(ByteBuffer message) {
        return message;
    }

    public ByteBuffer decodeMessage(ByteBuffer message) {
        return message;
    }
}
{% endhighlight %}

可以看到 = = 这玩意儿啥也没干。它充当了一个数据保持者的角色，在传递内存数据块时在编解码阶段免于内存拷贝。

#### JSONMessageCodec

先看`encodeMessage`方法：

{% highlight java%}

    public ByteBuffer encodeMessage(Object message) {
        if (message == null) {
            return null;
        } else {
            Object wrapped = JSONUtil.wrap(message);
            return wrapped instanceof String ? StringCodec.INSTANCE.encodeMessage(JSONObject.quote((String)wrapped)) : StringCodec.INSTANCE.encodeMessage(wrapped.toString());
        }
    }
    
{% endhighlight %}

里面使用JSONUtil.wrap方法打包message，然后调用了StringCodec的encodeMessage方法。

再看一下`decodeMessage`方法

{% highlight java%}

    public Object decodeMessage(ByteBuffer message) {
        if (message == null) {
            return null;
        } else {
            try {
                String json = StringCodec.INSTANCE.decodeMessage(message);
                JSONTokener tokener = new JSONTokener(json);
                Object value = tokener.nextValue();
                if (tokener.more()) {
                    throw new IllegalArgumentException("Invalid JSON");
                } else {
                    return value;
                }
            } catch (JSONException var5) {
                throw new IllegalArgumentException("Invalid JSON", var5);
            }
        }
    }


{% endhighlight %}

这个逻辑也比较简单，基本就是一个反过程。

#### StandardMessageCodec

最重量级的一个codec，不过大部分都是重复的逻辑代码。

首先定义了一系列byte值，从0到13，涵盖了包括map、list在内的数据类型。

{% highlight java%}

    private static final byte NULL = 0;
    private static final byte TRUE = 1;
    private static final byte FALSE = 2;
    private static final byte INT = 3;
    private static final byte LONG = 4;
    private static final byte BIGINT = 5;
    private static final byte DOUBLE = 6;
    private static final byte STRING = 7;
    private static final byte BYTE_ARRAY = 8;
    private static final byte INT_ARRAY = 9;
    private static final byte LONG_ARRAY = 10;
    private static final byte DOUBLE_ARRAY = 11;
    private static final byte LIST = 12;
    private static final byte MAP = 13;

{% endhighlight %}

来看一看`encodeMessage方法`和里面主要起作用的`writeValue`方法

{% highlight java%}

public ByteBuffer encodeMessage(Object message) {
        if (message == null) {
            return null;
        } else {
            StandardMessageCodec.ExposedByteArrayOutputStream stream = new StandardMessageCodec.ExposedByteArrayOutputStream();
            //这里将message写入stream
            this.writeValue(stream, message);
            ByteBuffer buffer = ByteBuffer.allocateDirect(stream.size());
            //从0开始，stream内容放进去
            buffer.put(stream.buffer(), 0, stream.size());
            return buffer;
        }
    }
    
    protected void writeValue(ByteArrayOutputStream stream, Object value) {
        if (value == null) {
            //stream第一个位置代表类型。
            stream.write(0);
        }
   
        //略.....
        
        else if (value instanceof byte[]) {
            stream.write(8);
            writeBytes(stream, (byte[])((byte[])value));
        } else {
            int var5;
            int var6;
            if (value instanceof int[]) {
                stream.write(9);
                int[] array = (int[])((int[])value);
                writeSize(stream, array.length);
                writeAlignment(stream, 4);
                int[] var4 = array;
                var5 = array.length;

                for(var6 = 0; var6 < var5; ++var6) {
                    int n = var4[var6];
                    //这里的方法实现原理是n右移8位，然后存
                    //在存Double这样的时候，先用Double.doubleToLongBit数字转成bit值，然后调用对应位数的写整形值方法
                    writeInt(stream, n);
                }
            } 
            
            //略....
            
            } else {
                //这里递归调用了writeValue方法，利用迭代器一个个的写进去。
                Iterator var15;
                if (value instanceof List) {
                    //先保存类型
                    stream.write(12);
                    List<?> list = (List)value;
                    //再保存大小
                    writeSize(stream, list.size());
                    var15 = list.iterator();
                    //再保存值。
                    while(var15.hasNext()) {
                        Object o = var15.next();
                        this.writeValue(stream, o);
                    }
                }
            }
    }

{% endhighlight %}

#### StringCodec

就很简单了，UTF8编码，存取。

### MethodCodec

好多逻辑都是通的，而且都是基于对应的MessageCodec。

{% highlight java%}

    ByteBuffer encodeMethodCall(MethodCall var1);

    MethodCall decodeMethodCall(ByteBuffer var1);

    ByteBuffer encodeSuccessEnvelope(Object var1);

    ByteBuffer encodeErrorEnvelope(String var1, String var2, Object var3);

    Object decodeEnvelope(ByteBuffer var1);

{% endhighlight %}

多了三个方法：

* `encodeSuccessEnvelope()`:方法调用成功
* `encodeErrorEnvelope()`:方法调用失败
* `decodeEnvelope()`:用于获取返回结果用。比如Android平台通过MethodChannel调用Flutter中的方法,且获取其返回结果


再看一下MethodCall类，每一个方法调用都被封装成了这个类。

{% highlight java%}

public final String method;
public final Object arguments;

{% endhighlight %}


两个实现方法：

![](https://i.loli.net/2019/03/16/5c8cb0f07a3a3.jpg)

两个方法会将method和args依次封装，原理和上面类似，就不再写啦。

















