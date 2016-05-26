---
layout: post
title:  "Android中TextView控件自动滚动"
author: 程鹏飞
tags: [Android, TextView, Scroll]
category: Android
---

在日常的软件开发中，程序员经常会使用到Log打印来测试程序的运行状态。一般的做法是直接使用android.util.Log提供的方法来在IDE中的控制台来打印，但是如果是在同一个电脑上调试多个程序，或者程序调试的时候不方便连接电脑，这样来回的切换便非常的不方便。如果在每个手机的界面上，模拟IDE控制台那样进行打印并自动向下滚动，这样便非常的方便直观。

####Layout布局文件####
	<TextView
        android:id="@+id/txt_log"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:singleLine="false"		      //多行显示
        android:scrollbars="vertical"		  //垂直方式滚动
        android:fadeScrollbars="false"/>     //一直显示滚动条
        
        
####Activity代码实现####
{% highlight java %}

//实例化控件，添加滑动事件  
TextView txt_log = (TextView)findViewById(R.id.txt_log);
txt_log.setMovementMethod(new ScrollingMovementMethod());


//将此代码添加到txt_log追加文字完成后的后面  
//存在txt_log.getLayout()返回的值为null，所以需要为null判断  
if(txt_log.getLayout() != null){  
   final int scrollAmount = txt_log.getLayout().getLineTop(txt_log.getLineCount()) - txt_log.getHeight();  
   //获取当前字体的高、宽
   Paint paint = new Paint();  
   paint.setTextSize(txt_log.getTextSize());  
   Paint.FontMetrics fm = paint.getFontMetrics();  
   int fontHeight =  (int) Math.ceil(fm.descent - fm.top);  
   //满足滚动条件，则滚动文字到底部  
   if(scrollAmount > fontHeight)
       txt_log.scrollTo(0,scrollAmount);
   }

{% endhighlight %}
