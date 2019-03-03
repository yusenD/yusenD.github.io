---
layout: post
title: Activity详解
feature-img: "assets/img/background/pexels-photo10.jpeg"
thumbnail: "assets/img/background/pexels-photo10.jpeg"
tags: [Android,面试]
---

> 自从我第一次认识你，以后我走的每一步都是为了接近你。 <br>
> <p align="right">——《艺伎回忆录》</p> 
* TOC
{:toc}

# Activity

Activity是Android四大组件之一，也是和用户交互的接口，提供了一个页面让用户进行滑动、点击等操作。

## 一、Activity生命周期

### 1.activity的4种状态

* running：处于活动状态，在栈顶，用户点击、滑动并响应
* paused：失去焦点的时候，被非全屏的activity、透明的activity覆盖，失去交互能力，对屏幕操作没有响应。在内存紧张的时候，会被回收
* stopped：被覆盖的activity就进入这个状态，会保存信息，成员变量
* killed：被系统回收掉了

### 2.activity生命周期分析

**activity启动的时候：**onCreate()->onStart()->onResume()

* onCreate：设置setContentView，数据的加载
* onStart：已经处于用户可见状态，但是还不能进行交互
* onResume：activity前台可见，可以进行交互

**点击home键回到主界面（activity不可见）：**onPause()->onStop()
* onPause()：整个activity是停止状态，可见但是不可交互
* onStop()：activity完全不可见

**再次回到activity：**onRestart()->onStart()->onResume()
* onRestart：再次回来的时候执行

**退出当前activity：**onPause()->onStop()->onDestroy()
* onDestroy：做一些资源回收释放

### 3.Android进程优先级

* 前台：处于与用户交互的activity或者与交互activity绑定的service
* 可见：不可点击的
* 服务：后台开启的service
* 后台：按了home键，不可见时
* 空：不属于前面四种，可以随时kill掉

## 二、Android任务栈和Activity启动模式

很重要，task，用栈存储activity，系统可以通过任务栈管理每一个activity，任务栈和启动模式息息相关。

* standard：标准模式，每创建一个activity都会执行一遍onCreate等流程，耗费内存资源
* singleTop：栈顶复用模式，判断任务栈顶中有没有这个activity，如果没有在栈顶仍然重新创建
* singleTask：栈内复用模式，单例，检测整个任务栈内是否存在要创建的activity，如果存在，就置于栈顶并且在他之上的所有activity全部出栈
* singleInstance：如果在整个系统中有且只有一个实例，activity独享任务栈

### 常用场景

## scheme跳转协议

