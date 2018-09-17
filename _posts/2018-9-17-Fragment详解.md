---
layout: post
title: Fragment详解（二）
feature-img: "assets/img/background/qingdao-street.jpg"     
thumbnail: "assets/img/thumbnails/pexels-photo1.jpeg"
tags: [Android,fragment]
---

Fragment依赖Activity，下面将Fragment的生命周期和Activity联合起来看。

<br>
![](https://i.loli.net/2018/09/17/5b9f149bcd634.jpg)

<br>

我们这里举个例子，有F1和F2两个Fragment，在初始化的时候加入Activity，点击F1中的按钮调用`replace()`方法替换成F2。

Log如下：

```

BasicActivity: [onCreate] BEGIN
BasicActivity: [onCreate] END
BasicActivity: [onStart] BEGIN
Fragment1: [onAttach] BEGIN 
Fragment1: [onAttach] END
BasicActivity: [onAttachFragment] BEGIN
BasicActivity: [onAttachFragment] END
Fragment1: [onCreate] BEGIN
Fragment1: [onCreate] END
Fragment1: [onCreateView]
Fragment1: [onViewCreated] BEGIN
Fragment1: [onViewCreated] END
Fragment1: [onActivityCreated] BEGIN
Fragment1: [onActivityCreated] END
Fragment1: [onStart] BEGIN
Fragment1: [onStart] END
BasicActivity: [onStart] END
BasicActivity: [onPostCreate] BEGIN
BasicActivity: [onPostCreate] END
BasicActivity: [onResume] BEGIN
BasicActivity: [onResume] END
BasicActivity: [onPostResume] BEGIN
Fragment1: [onResume] BEGIN
Fragment1: [onResume] END
BasicActivity: [onPostResume] END
BasicActivity: [onAttachedToWindow] BEGIN
BasicActivity: [onAttachedToWindow] END

```

从上面我们可以看到：


* Fragment的onActtach => onCreate => onCreateView => onActivityCreated => onStart 都是在Activity中的onStart调用的。
* Fragment的onResume在Activity的onResume之后调用

然后，当加上addToBackStack和不加addToBackStack两种情况：

1.点击按钮，用replace方法替换成F2，并且不加addToBackStack，日志如下：


```

Fragment2: [onAttach] BEGIN
Fragment2: [onAttach] END
BasicActivity: [onAttachFragment] BEGIN
BasicActivity: [onAttachFragment] END
Fragment2: [onCreate] BEGIN
Fragment2: [onCreate] END
Fragment1: [onPause] BEGIN
Fragment1: [onPause] END
Fragment1: [onStop] BEGIN
Fragment1: [onStop] END
Fragment1: [onDestroyView] BEGIN
Fragment1: [onDestroyView] END
Fragment1: [onDestroy] BEGIN
Fragment1: [onDestroy] END
Fragment1: [onDetach] BEGIN
Fragment1: [onDetach] END
Fragment2: [onCreateView]
Fragment2: [onViewCreated] BEGIN
Fragment2: [onViewCreated] END
Fragment2: [onActivityCreated] BEGIN
Fragment2: [onActivityCreated] END
Fragment2: [onStart] BEGIN
Fragment2: [onStart] END
Fragment2: [onResume] BEGIN
Fragment2: [onResume] END

```
可以看到F1最后调用了`onDestory`和`onDetach`


2.当点击F1的按钮，调用replace替换成F2，并且加addToBackStack时，日志如下：


```

Fragment2: [onAttach] BEGIN
Fragment2: [onAttach] END
BasicActivity: [onAttachFragment] BEGIN
BasicActivity: [onAttachFragment] END
Fragment2: [onCreate] BEGIN
Fragment2: [onCreate] END
Fragment1: [onPause] BEGIN
Fragment1: [onPause] END
Fragment1: [onStop] BEGIN
Fragment1: [onStop] END
Fragment1: [onDestroyView] BEGIN
Fragment1: [onDestroyView] END
Fragment2: [onCreateView]
Fragment2: [onViewCreated] BEGIN
Fragment2: [onViewCreated] END
Fragment2: [onActivityCreated] BEGIN
Fragment2: [onActivityCreated] END
Fragment2: [onStart] BEGIN
Fragment2: [onStart] END
Fragment2: [onResume] BEGIN
Fragment2: [onResume] END

```

可以看到，只调用了`onDestoryView`，并没有调用`onDestory`和`onDetach`。




