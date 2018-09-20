---
layout: post
title: Fragment详解（四）
feature-img: "assets/img/background/pexels-photo1.jpeg"
thumbnail: "assets/img/thumbnails/pexels-photo3.jpeg"
tags: [Android,Fragment]
---

Fragment的基础知识了解的差不多了，下面了解一下Fragment的通信。

Fragment的通信主要有三部分：
* Fragment向Activity传数据
* Activity向Fragment传数据
* Fragment之间互传数据
<br>


## Fragment向Activity传数据
----


可以在Fragment中定义接口，并且让Activity实现该接口，然后在Fragment的onAttach的方法中，将context转成接口对象，然后调用即可。

举个例子🌰：

接口定义：

{% highlight java%}
public interface OnFragmentListener{
    void onItemClick(String str);
}
{% endhighlight %}


Activity实现接口：

{% highlight java%}
public class Activity extends Activity implements OnFragmentListener {

    //Something

    @Override
    public void onItemClick(String str){
        textView.setText(str);
    }

}
{% endhighlight %}


Fragment调用
{% highlight java%}
public class F1 extends Fragment {
    @Override
    public void onAttach(Activity activity){
        super.onAttach(activity);
        mListener = (OnFragmentListener) activity;
    }

    @Override
    public void onClick(View v){
        //Do Something
        mListener.onItemClick("hi activity");
    }

}
{% endhighlight %}

<br>


## Activity向Fragment传递数据



