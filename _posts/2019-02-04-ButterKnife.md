---
layout: post
title: ButterKnife原理解析
feature-img: "assets/img/background/pexels-photo5.jpeg"
tags: [Android,面试]
---

> 最近好丧。唉。                         


# ButterKnife

依赖注入框架，Android之神写的。巧妙地用了Java的APT（注解处理器），避免了Runtime中使用大量反射，造成系统性能降低。

## Java反射

## Java注解

## ButterKnife原理

* 编译的时候扫描注解，并做相应处理，生成java代码（调用了javapoet库生成）
* 调用Butterknife.bing(this)时候，将ID于对应的上下文绑定在一起

### ButterKnifeProcessor

之所以init是同步方法，是为了保证单例，只会在初始的时候调用一次，传入了ProcessEnvironment工具类参数。

* findAndParseTargets 方法


