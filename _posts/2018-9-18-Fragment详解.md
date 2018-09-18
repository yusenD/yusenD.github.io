---
layout: post
title: Fragment详解（三）
feature-img: "assets/img/background/qingdao-street.jpg "
thumbnail: "assets/img/thumbnails/pexels-photo2.jpeg"
tags: [Android,Fragment]
---

Activity有任务栈，用户通过`startActivity`来将Activity入栈，点击返回按钮将Activity出栈。Fragment也有类似的栈，称为Back Stack（回退栈）。

Back Stack由FragmentManager管理，在默认的情况下，Fragment事务不会加入回退栈，如果想将Fragment事务加入回退栈，可以用`addToBackStack("")`。如果没有加入回退栈的话，用户点击返回按钮会直接将Activity出栈；而如果加入了回退栈，用户点击返回按钮会回滚Fragment事务。

下面👇举个🌰：

```
getSupportFragmentManager().beginTransaction()
    .add(R.id.container,"f1")
    .addToBackStack("")
    .commit();
```

上面的代码功能是将Fragment加入到Activity中，内部的实现为：

创建BackStackRecord对象，该对象交代了这个事务的全部操作轨迹（即 add 操作）,然后将该对象提交到FragmentManager的执行队列中，

