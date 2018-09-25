---
layout: post
title: Fragment详解（三）
feature-img: "assets/img/background/pexels-photo1.jpeg "
thumbnail: "assets/img/thumbnails/pexels-thumbnail1.jpeg"
tags: [Android,Fragment]
---

> *凛冽而热血*


Activity有任务栈，用户通过`startActivity`来将Activity入栈，点击返回按钮将Activity出栈。Fragment也有类似的栈，称为Back Stack（回退栈）。


<br><br>


## 通过Fragment源码看原理
----
Back Stack由FragmentManager管理，在默认的情况下，Fragment事务不会加入回退栈，如果想将Fragment事务加入回退栈，可以用`addToBackStack("")`。如果没有加入回退栈的话，用户点击返回按钮会直接将Activity出栈；而如果加入了回退栈，用户点击返回按钮会回滚Fragment事务。

下面👇举个🌰：

```
getSupportFragmentManager().beginTransaction()
    .add(R.id.container,"f1")
    .addToBackStack("")
    .commit();
```

上面的代码功能是将Fragment加入到Activity中，内部的实现为：

创建BackStackRecord对象，该对象交代了这个事务的全部操作轨迹（即 add 操作），然后将该对象提交到FragmentManager的执行队列中，等待执行。

`BackStackRecord`类的定义如下：
{% highlight java%}
class BackStackRecord extends FragmentTransaction implements FragmentManager.BackStackEntry, Runnable {}
{% endhighlight %}

可以看到，BackStackRecord有三重定义：
* 继承了FragmentTransaction，即是事务，保存了整个事务的全部操作轨迹。
* 实现了BackStackEntry，作为回退栈的元素，正是因为该类拥有事务全部的操作轨迹，因此在`popBackStack()`时能回退整个事务。
* 继承了Runnable，即被放入FragmentManager执行队列，等待被执行。

先看第一层含义，`getSupportFragmentManager.beginTransaction()`返回的就是BackStackRecord对象，代码如下：

{% highlight java%}
public FragmentTransaction beginTransaction(){
    return new BackStackRecord(this);
}
{% endhighlight %}

BackStackRecord类包含了一次事务的整个操作轨迹，是以链表形式存在的，链表的元素是OP类，表示其中某个操作，定义如下：

{% highlight java%}
static final class Op {
    Op next;   //链表的后一个节点
    Op prev;   //链表的前一个节点
    int cmd;   //操作类型 add、remove、replace等
    Fragment fragment; //对那个Fragment进行操作
}
{% endhighlight %}

看一下具体场景下面，这些类都是怎么使用的，以add为例：

{% highlight java%}
public FragmentTransaction add(int containerViewId,Fragment fragment,String tag){
    doAddOp(containerViewId,fragment,tag,op_ADD);
    return this;
}
{% endhighlight %}


`doAddOp()`方法就是创建Op对象，并且加入链表，定义如下：

{% highlight java%}
private void doAddOp(int containerViewId,Fragment fragment,String tag,int opcmd){
    fragment.mTag = tag; //设置Fragment的tag
    fragment.mContainerId = fragment.mFragmentId = containerViewId; //设置fragment的容器Id
    Op op = new Op();
    op.cmd = opcmd;
    op.fragment = fragment;
    addOp(op); //将创建好的对象加入链表
}
{% endhighlight %}

`addToBackStack()`方法是将mAddToBackStack变量记为true，在`commit()`中会用到该变量。`commit()`是异步的，并不是立即生效的，但是后面会看到整个过程还是在主线程完成的，只是把事务的执行扔给了主线程的Handler，`commit()`内部是`commitInternal()`，实现如下：

{% highlight java%}
int commitInternal(boolean allowStateLoss) {
    mCommitted = true;
    if(mAddToBackStack){
        mIndex = mManager.allocBackStackIndex(this);
    }else{
        mIndex = -1;
    }
    mManager.enqueueAction(this,allowStateLoss);
    return mIndex;
}
{% endhighlight %}

如果mAddToBackStack为true，则调用`allocBackStackIndex()`方法将事务添加进回退栈，FragmentManager类的变量`ArrayList<BackStackRecord>`就是回退栈。实现如下：

{% highlight java%}
public int allocBackStackIndex(BackStackRecord bse) {
    if (mBackStackIndices == null) {
        mBackStackIndices = new ArrayList<BackStackRecord>();
    }
    int index = mBackStackIndices.size();
    mBackStackIndices.add(bse);
        return index;
}
{% endhighlight %}

在`commitInternal()`中，`mManager.enqueueAction(this,allowStateLoss)` 是将BackStackRecord加入待执行队列中，定义如下：

{% highlight java%}
public void enqueueAction(Runnable action,boolean allowStateLoss) {
    if(mPendingActions == null){
        mPendingActions = new ArrayList<Runnable>();
    }
    mPendingActions.add(action);
    if(mPendingActions.size()==1){
        mHost.getHandler().removeCallbacks(mExecCommit)l;
        mHost.getHandler().post(mExecCommit);
    }
}
{% endhighlight %}

mPendingActions就是待执行队列，`mHost.getHandler()`就是主线程的Handler，因此Runnable是在主线程执行的，mExecCommit的内部就是调用了`execPendingActions()`，即把mPendingActions中所有积压的没被执行的事务全部执行。执行队列中的事务会调用`BackStackRecord  run()`方法来执行，执行Fragment的生命周期函数，将视图添加到container中。

与`addToBackStack()`对应的是`popBackStack()`，有以下几种变种：

* popBackStack()：将回退栈的栈顶弹出，并且回退该事务。
* popBackStack(String name,int flag)：name是`addToBackStack(String name)`的参数，通过name可以找到回退栈的特定元素，flag可以为0或者FragmentManager.POP_BACK_STACK_INCLUSIVE，前者表示之弹出该元素之上的特定元素，FragmentManager.POP_BACK_STACK_INCLUSIVE表示弹出包括该元素及以上的元素。
* 这个方法执行是异步执行的，是丢到主线程的MessageQueue执行，popBackStackImmediate()是同步版本。

**另外**，`findFragmentByTag()`是经常用到的方法，他是FragmentManager的方法，FragmentManager是抽象类，FragmentManagerImpl是继承FragmentManager的实现类，它的内部实现是：

{% highlight java%}
class FragmentManagerImpl extends FragmentManager {
    ArrayList<Fragment> mActive;
    ArrayList<Fragment> mAdded;
    public Fragment findFragmentByTag(String tag) { 
           if (mAdded != null && tag != null) { 
               for (int i=mAdded.size()-1; i>=0; i--) {
                Fragment f = mAdded.get(i);
                if (f != null && tag.equals(f.mTag)) {
                        return f;
                }
            }
        }       
          if (mActive != null && tag != null) {
               for (int i=mActive.size()-1; i>=0; i--) {
                    Fragment f = mActive.get(i);
                    if (f != null && tag.equals(f.mTag)) {
                          return f;
                }
            }
        } 
          return null;
    }
}
{% endhighlight %}


从上面可以看到,先从mAdded中查找是否有该Fragment，如果没有找到，再从mActvite中查找是否有该Fragment。mAdded是已经添加到Activity的Fragment集合，mActive不仅包含mAdded，而且还包含虽然不在Activity中，但是还在回退栈的Fragment。
<br><br>
## 举个例子🌰
----
假设现在有三个Fragment：F1,F2,F3。F1初始化就加入Activity，点击F1按钮跳转到F2，点击F2按钮跳转到F3，最后点击F3跳转到F1。

这样做：

在Activity的onCreate中，将F1加入Activity：

{% highlight java%}
getSupportFragmentManager().beginTransaction()
    .add(R.id.container,f1,"f1")
    .addToBackStack(F1.class.getSimpleName())
    .commit();
{% endhighlight %}

F1的按钮点击事件如下：

{% highlight java%}
getFragmentSupportManager().beginTransaction()
    .replace(R.id.container,f2,"f2")
    .addToBackStack(F2.class.getSimpleName())
    .commit();
{% endhighlight %}

F2的按钮点击事件如下：

{% highlight java%}
getFragmentSupportManager().beginTransaction()
    .replace(R.id.container,f3,"f3")
    .addToBackStack(F2.class.getSimpleName())
    .commit();
{% endhighlight %}


F3的按钮点击事件如下：

{% highlight java%}
getFragmentSupportManager().popBackStack(F2.class.getSimpleName(),FragmentManager.POP_BACK_STACK_INCLUSIVE);
{% endhighlight %}

这样就完成了整个页面的跳转逻辑。
<br>


<br>
> 参考公众号文章：腾讯Bugly



















