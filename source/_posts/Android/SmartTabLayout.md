layout: post
title: "SmartTabLayout的架构分析"
excerpt: "详细分析SmartTabLayout的结构和设计方法."
tags: 
- SmartTabLayout
- Android
categories:
- Android
---

<figure class="half">
	<a href="https://github.com/ogaclejapan/SmartTabLayout">Github链接</a>
</figure>

该文章主要分析各个类的结构，以及他们之间的关系，重点考察其设计模式，封装思想。   


{% img https://raw.githubusercontent.com/ogaclejapan/SmartTabLayout/master/art/demo1.gif %}

{% img https://raw.githubusercontent.com/ogaclejapan/SmartTabLayout/master/art/demo2.gif  [效果图]%}

<!-- more --> 

---

#### 主要分析点  

* 视图分析
* SmartTabLayout的结构分析
* TabStrip的结构分析
* 适配的重新封装
* 设计模式的运用以及合理重新封装


#### 视图分析
{% img /images/SmartTabLayout_Analysis/SmartTabLayout视图分析.png  [SmartTabLayout视图分析]%}

由表及里，根据视图来分析代码的执行顺序

1. 初始化SmartTabLayout
2. 将SmartTabStrip添加到SmartTabLayout中

此时，用户调用SetViewPager方法

{% codeblock [SetViewPager] %}
	public void setViewPager(ViewPager viewPager) {
	        tabStrip.removeAllViews();
	        this.viewPager = viewPager;
	        if (viewPager != null && viewPager.getAdapter() != null) {
	            viewPager.addOnPageChangeListener(new InternalViewPagerListener()); //监听ViewPager的滑动
	            populateTabStrip();//填充TabStrip
	        }
	}
{% endcodeblock %}

填充TabStrip就是，我们创建一个个TabView，将其添加到TabStrip中。用户可以自定义TabView，也可以用默认的TabView进行填充。

接着,在TabStrip的onDraw()中，将Decoration绘制出来,那么，静态的视图就出来了

此处,还有一个梗，此开源框架给用户提供了一种选项：isIndicatorAlwaysInCenter
即选中的Tab始终居中，那么，我们需要在某个回调方法中对视图进行一些调整。  
通过设置SmartTabLayout的padding值来调整初始位置

{% codeblock [onSizeChanged] %}
    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        if (tabStrip.isIndicatorAlwaysInCenter() && tabStrip.getChildCount() > 0) {
            ViewCompat.setPaddingRelative(this, start, getPaddingTop(), end, getPaddingBottom());
            setClipToPadding(false); //很关键的一句代码
        }
    }
{% endcodeblock %}

如果没有setClipToPadding(false),那么子视图的画布是不包括padding的，那么SmartTabStrip是无法滑动到父视图的最左端的

#### SmartTabLayout类的分析

* SmartTabLayout类结构图  

{% img /images/SmartTabLayout_Analysis/smart_tab_layout_01.png  [SmartTabLayout类结构图]%}

作为暴露给用户的类,提供了各种可配置的接口,如ViewPager滑动的监听器,CustomTabView,TabTextColor等各种参数和配置。
在内部，它将这些attrs传给了SmartTabStrip，用于生成对应的控件


* SmartTabStrip类结构图  

{% img /images/SmartTabLayout_Analysis/smart_tab_layout_02.png  [SmartTabStrip类结构图]%}

SmartTabStrip作为TabView的容器,同时负责将Decoration绘制出来，它从SmartTabLayout中拿到滑动的距离,然后选择某种方式将Decoration绘制出来

---

#### 流程图

  {% img /images/SmartTabLayout_Analysis/SmartTabLayout流程图.png  [流程图]%}

  根据SmartTabLayout的类结构分析，它内部包含TabStrip类，即他们是组合关系    

  1. 将TabStrip对象创建出来，并添加到SmartTabLayout对象中
  2. 将SmartTabLayout与ViewPager进行绑定  
  3. 填充TabStrip，根据用户是否设置了TabProvider动态的创建TabView  
  4. 布局TabStrip,根据distributeEvenly参数动态的进行布局  
  5. 在onLayout完成之后适当的进行scrollTo操作，以便达到合适的显示效果（indicatorAlwaysInCenter参数）  
  
#### 用户交互流程
  
  当用户操作SmartTabLayout时，它是怎样进行交互的  
  1. **用户对ViewPager进行操作**  
     SmartTabLayout的内部类InternalViewPagerListener实现了对ViewPager的监听，当滑动事件发生后，做两件事情  

     1.通知SmartTabStrip滑动事件发生，并将滑动的position和positionOffset传给他  
     2.调用自身的scrollToTab方法

  2. **对SmartTabLayout进行滑动操作**
     由于SmartTabLayout继承自HorizontalScrollView,滑动操作会直接调用父类的scrollTo方法

  3. **对TabView进行点击操作**
     直接跳转到该ViewPager


#### 设计模式的运用以及合理重新封装

  该类是策略模式的体现，即SmartTabLayout提供机制（接口），而SmartTabStrip提供策略（具体实施者）。
  SmartTabLayout负责提供绘制参数，他作为SmartTabStrip的控制者，并在用户滑动的时候调用SmartTabStrip进行重绘。  
  模块化的思想，将TabView设计成一个模块，在SmartTabStrip中，对Decoration的绘制也是动态的，可以以下划线的形式出现，也可以是矩形块的形式出现  
  SmartTabLayout负责提供接口，监听接口，自定义的TabView等等，拿到所有的配置属性，而SmartTabStrip是实施者，真正绘制图形的类  

     