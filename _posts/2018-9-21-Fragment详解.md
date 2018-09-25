---
layout: post
title: Fragment详解（五）
feature-img: "assets/img/background/pexels-photo3.jpeg"     
thumbnail: "assets/img/thumbnails/pexels-thumbnail3.jpeg"
tags: [Android,Fragment]
---

Fragment最常使用的场景，ViewPager + Fragment就不得不说了。很多APP的首页都是这么实现的。下面看一下优雅的使用方式。
<br>


## 基本使用
----

「ViewPager」 是support v4库中提供的界面滑动的类，继承自ViewGroup。

「PagerAdapter」 是ViewPager的适配器类，为ViewPager提供界面，但是一般都是使用PagerAdapter的两个子类：`FragmentPagerAdapter`和`FragmentStatePagerAdapter`作为ViewPager的适配器，区别如下：

**FragmentPagerAdapter：**
>   Implementation of PagerAdapter that represents each page as a Fragment that is persistently kept in the fragment manager as long as the user can return to the page.
This version of the pager is best for use when there are a handful of typically more static fragments to be paged through, such as a set of tabs. The fragment of each page the user visits will be kept in memory, though its view hierarchy may be destroyed when not visible. This can result in using a significant amount of memory since fragment instances can hold on to an arbitrary amount of state. For larger sets of pages, consider FragmentStatePagerAdapter.

**FragmentStatePagerAdapter：**
> Implementation of PagerAdapter that uses a Fragment to manage each page. This class also handles saving and restoring of fragment's state.
This version of the pager is more useful when there are a large number of pages, working more like a list view. When pages are not visible to the user, their entire fragment may be destroyed, only keeping the saved state of that fragment. This allows the pager to hold on to much less memory associated with each visited page as compared to FragmentPagerAdapter at the cost of potentially more overhead when switching between pages.


从官方文档的描述来看， 「FragmentPagerAdapter」 每一个生成的Fragment都将会保存在内存之中，适合数量较少，相对来说静态的页面；而 「FragmentStatePagerAdapter」 生成的Fragment，只保留当前页面，当页面不可见时候就会销毁，释放资源，所以在拥有大量的Fragment的时候，所占内存较少。

默认的，ViewPager会缓存当前页面的相邻页面，比如，在滑动到第二页的时候，会初始化第一页和第三页的Fragment界面（运行到`onResume()`），可以通过`setOffscreenPageLimit(count)`方法来设置缓存的界面个数。

两者需要重写的方法都一样，常用的如下：

* ` public FragmentPagerAdapter(FragmentManager fm) ` ：构造函数，参数是`FragmentManager`。如果是嵌套Fragment的场景，子PagerAdapter的参数传`getChildFragmentManager()`。
* ` Fragment getItem(int position) ` ：返回position位置的Fragment
* ` int getCount() ` ：返回ViewPager的页数
* ` Object instantiateItem(ViewGroup container,int position) ` ：container是ViewPager对象，返回第position位置的Fragment，预加载的时候会调用
* ` void destroyItem(ViewGroup container,int position,Object object) ` ：删除Fragment对象

<br>
## 懒加载
-----

懒加载主要用于ViewPager且每页都是Fragment的情况，比如微信等应用，当划到另一个tab的时候，显示正在加载，然后才显示正常的界面。

之前提到了，在默认情况下，ViewPager会缓存当前页面左右两边的页面，而很多情况下，用户看不到的页面有很多网络等耗时耗资源的操作，这是不必要的，所以采用懒加载。

懒加载的实现思路是：用户不可见的页面，只初始化UI，但是不会做任何数据加载等操作，等到滑动到该页面的时候，才会意不更新加载数据，刷新UI。

目标：实现类似微信的效果，当用户滑到另一个页面的时候，会显示正在加载的页面，等待数据加载完毕之后，显示正常的页面。

ViewPager默认缓存左右相邻的界面，避免不必要的数据加载（即重复调用onCreateView），因为有4个tab，因此将离线缓存设置为3，即` setOffscreenPageLimit(3) `。

懒加载依赖于Fragment的` setUserVisibleHint(boolean isVisible) `方法，当Fragment变为可见的时候，会调用` setUserVisibleHint(true) `；当Fragment变为不可见的时候，会调用` setUserVisibleHint(false) `，该方法调用时间：

* onAttach()之前，调用` setUserVisibleHint(false) `
* onCreateView之前，如果该页面为当前页，则为 「true」 ，否则为 「false」

举个例子🌰：

{% highlight java%}

public class DemoFragment extends Fragment {
    
    private boolean isInited;
    private boolean isPrepared;
    @Override
    public View onCreateView(LayoutInflater inflater,ViewGroup container,Bundle saveInstanceState){
        view = inflater.inflate(R.layout.fragment,container,false);
        isPrepared = true;
        lazyLoad();
        return view;
    }
    
    public void lazyLoad(){
        if(getUserVisibleHint() && !isInited && isPrepared){
            //异步初始化数据，数据加载完成之后更新UI
            loadData();
        }
    }
    
    private void loadData(){
        //异步加载数据
    }
    
    @Override
    public void setUserVisibleHint(boolean isVisibleToUser){
        super.setUserVisibleHint(isVisibleToUser);
        if(is VisibleToUser){
            lazyLoad();
        }
    }

}

{% endhighlight %}

需要注意的是：

* 在Fragment里面有两个变量` isPrepared `和` isInited `。
    * ` isPrepared ` ： 表示UI是否准备好了。因为数据加载之后需要更新UI，如果UI还没有inflate，就不需要做数据加载，因为` setUserVisibleHint() `会在` onCreateView() `之前调用一次，如果此时调用的话，UI还没有inflate，不能加载数据
    * ` isInited ` ： 表示是否已经做过数据加载。如果已经做过了就不需要再做了，因为setUserVisibleHint(true)在界面可见时都会调用，如果滑到该页面做过数据加载之后，切换，再切换回来，还是会调用` setUserVisibleHint(true) `，此时由于` isInited `为true，所以不会再做一遍数据加载。
* lazyLoad()：懒加载的核心，只有界面可见+UI准备好了+没有做过数据加载，才调用` loadData() `做数据加载，加载之后吧isInited设置为true。


<br><br>

> 参考公众号文章：腾讯Bugly

