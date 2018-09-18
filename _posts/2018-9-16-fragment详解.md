---
layout: post
title: Fragment详解（一）
feature-img: "assets/img/background/qingdao-street.jpg"     
thumbnail: "assets/img/thumbnails/desk-messy.jpeg"
tags: [Android,Fragment]
---

Fragment是开发中非常常用的控件，最近开发用了不少Fragment，搜集资料花了不少功夫，稍微总结一下


* TOC
{:toc}
<br>
## 概述
----
Fragment是在`API 11`（Android 3.0）加入的，后来为了兼容更低版本，开发了support-v4库的fragment。  
目前Fragment有`android.app.fragment`和`android.support.v4.app.Fragment`，在当前`API 28`（Android 9.0）中，前者已经不推荐使用了。  
所以更推荐使用后者。

![](https://i.loli.net/2018/09/16/5b9e4aaac4d0e.jpg)


<br>
**Fragment官方的定义是：**  

>A Fragment represents a behavior or a portion of user interface in an Activity. You can combine multiple fragments in a single activity to build a multi-pane UI and reuse a fragment in multiple activities. You can think of a fragment as a modular section of an activity, which has its own lifecycle, receives its own input events, and which you can add or remove while the activity is running.  


<br>
根据上面的定义可知：
* Fragment是依赖于Activity的，不能独立存在的。

* 一个Activity里可以有多个Fragment。

* 一个Fragment可以被多个Activity重用。

* Fragment有自己的生命周期，并能接收输入事件。

* 我们能在Activity运行时动态地添加或删除Fragment。
<br>

**Fragment的主要方法**  
* Fragment：Fragment的基类，任何创建fragment都需要继承自该类。
* FragmentManager：主要是负责管理和维护Fragment。是抽象类，具体实现类是FragmentManagerImpl。
* FragmentTransaction：对Fragment的添加、删除等操作都需要通过事务方式进行。是抽象类，具体的实现类BackStackRecord。
* NestedFragment：在Fragment的内部嵌套Fragment
<br><br>

## 生命周期
----
需要注意的是Fragment不是一个View，而是和Activity同一层次的。

Fragment的生命周期与Activity类似，但是更为复杂。


![](https://i.loli.net/2018/09/16/5b9e5e1940a2f.jpg)

每一步解释如下：

  onAttach() | Fragment和Activity相关联的时候进行调用，通过该方法可以获取Activity引用，还可以用`getArguments`来获取参数 
  onCreate() | Fragment被创建之后进行调用 
  onCreateView() | 创建Fragment的布局 
  onActivityCreate() | 当Activity完成onCreate()时调用
  onStart() | Fragment可见是进行调用
  onResume() | Fragment可见并且可以交互是进行调用
  onPause() | Fragment不可交互但可见调用
  onStop() | Fragment不可见是调用
  onDestoryView() | Fragment的UI从视图结构中移除时调用
  onDestory() | 销毁Fragment时调用
  onDetach() | Fragment和Activity解除关联时调用

以上方法，出了onCreateView在重写的时候不用写super方法，其余的都需要。

<br><br>

## 使用
----
### 1.基本用法
<br>
{% highlight java %}
public class Fragment1 extends Fragment{  
    
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.hello_fragment, container, false);
        return view;
    }
    
    public static Fragment1 newInstance(String str) {
        Fragment1 frag = new Fragment1();
        Bundle bundle = new Bundle();
        bundle.putString(ARG_PARAM, str);
        fragment.setArguments(bundle);   //设置参数
        return fragment;
    }    
    
}
{% endhighlight %}

<br>
重写onCreateView方法，绑定layout，页面就有了。

如果要传递参数的话，可以通过`setArguments`方法传入`Bundle`，不推荐用构造方法的方法，因为通过setArguments()方式添加，在由于内存紧张导致Fragment被系统杀掉并恢复（re-instantiate）时能保留这些数据。

官方建议：
>It is strongly recommended that subclasses do not have other constructors with parameters, since these constructors will not be called when the fragment is re-instantiated.

如果要获取到这些参数，可以用Fragment的`onAttach`方法通过`getArguments`方法获得，并且在获取之后使用这些参数。  

如果要获取的Activity对象，不建议用`getActivity`方法，而是在`onAttach`方法中将Context对象强转成Activity对象。

{% highlight java %}
private Activity mActivity;
private String param;
@Override
public void onAttach(Context context) {
    mActivity = (Activity)context;
    param = getArguments().getString(KEY_PARAM);
}
{% endhighlight %}

<br>
### 2.Fragment添加Activity
<br>
主要有两种方法：
* 静态添加：  直接在xml文件中进行绑定，非常不灵活，很少用到
* 动态添加：  在代码中添加，比较灵活，建议使用

在Activity的XML文件加入一个布局容器，最简单的FrameLayout即可，然后通过下面代码加入到Activity中：
```
getSupportFragmentManager().beginTransaction()
    .add(R.id.fl_container, HelloFragment.newInstance("hello world"), "f1")
    .commit();
```

**需要注意的是：**  

* 因为使用的是support-v4库的Fragment，所以需要使用`getSupportFragmentManager`方法来获取FragmentManager。
* `add()`方法中有三个参数，第一个是容器，第二个是Fragment对象，第三个是Fragment的「 tag 」名，作用是以后可以用`findFragmentByTag(tag)`方法，根据tag从FragmentManager中找到Fragment对象。
* 在一次事务中，可以进行多个操作，比如`add().remove().replace()`
* `commit()`操作是异步的，内部通过`enqueueAction()`加入处理队列。对应的同步方法为`commitNow()`，`commit()`内部会有`checkStateLoss()`操作，如果开发人员使用不当（比如commit()操作在onSaveInstanceState()之后），可能会抛出异常，而`commitAllowingStateLoss()`方法则是不会抛出异常版本的`commit()`方法，但是尽量使用`commit()`，而不要使用`commitAllowingStateLoss()`。
* `addToBackStack("fname")`是可选的。FragmentManager拥有回退栈（BackStack），类似于Activity的任务栈，如果添加了该语句，就把该事务加入回退栈，当用户点击返回按钮，会回退该事务（回退指的是如果事务是add(frag1)，那么回退操作就是remove(frag1)）；如果没添加该语句，用户点击返回按钮会直接销毁Activity。
* Fragment有一个常见的问题，即Fragment重叠问题，这是由于Fragment被系统杀掉，并重新初始化时再次将fragment加入activity，因此通过在外围加if语句能判断此时是否是被系统杀掉并重新初始化的情况。

<br>
### 3.常见异常
<br>

如下：
```
java.lang.IllegalStateException: Can not perform this action after onSaveInstanceState
    at android.support.v4.app.FragmentManagerImpl.checkStateLoss(FragmentManager.java:1341)
    at android.support.v4.app.FragmentManagerImpl.enqueueAction(FragmentManager.java:1352)
    at android.support.v4.app.BackStackRecord.commitInternal(BackStackRecord.java:595)
    at android.support.v4.app.BackStackRecord.commit(BackStackRecord.java:574)
```

该异常出现的原因是：`commit()`在`onSaveInstanceState()`后调用。首先，`onSaveInstanceState()`在`onPause()`之后，`onStop()`之前调用。`onRestoreInstanceState()`在`onStart()`之后，`onResume()`之前。

因此避免出现该异常的方案有：

* 不要把Fragment事务放在异步线程的回调中，比如不要把Fragment事务放在AsyncTask的onPostExecute()，因此onPostExecute()可能会在onSaveInstanceState()之后执行。

* 逼不得已时使用commitAllowingStateLoss()。




<br>
<br>
> 参考公众号：腾讯Bugly



