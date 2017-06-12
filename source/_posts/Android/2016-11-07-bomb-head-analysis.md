---
layout: post
title: "Bomb Head Analysis"
excerpt: "Bomb Head动画分析"
tags: 
- animation
- Android
categories:
- Android
comments: true
share: true
---


在完成了<a href="http://www.pengjiebest.com/articles/store-house-animation-analysis/">StoreHouse动画分析</a>之后,想进一步巩固View动画方法的知识,正好又看到了一个不错的gif动画，两者摩擦出了行动的火花，遂有此文。

<!-- more --> 

原gif图如下：
![](http://i.imgur.com/zREtq7h.gif)

看到这个动画的时候，我就想把他做成一个下拉刷新的头部，既不是太难，也可以很有挑战性。

### 主要模块

* 整体思路分析
* 下拉时的移动
* 刷新时的动画
* 具体代码分析

#### 整体思路分析

**在哪里做动画**

在此之前，梳理一下View做动画的思路

 1. 给View设置动画对象  
 	 View.setAnimation(new ScaleAnimation())
 2. 给动画对象添加监听器  
	 Animation.setAnimationListener()
 3. 开启动画  
 	 Animation.start()

也就是说，这个动画是针对整个View对象的，更具体点说，是针对View画布上所画的所有东西。假如你在View上只画一个圆，对这个View使用ScaleAnimation，那么画的圆就会做出动画效果。  
可是，现在这不只有一个圆，如果要把动画效果加在View上，那么这些圆都会执行同样的动画，这显然不是我们想要的结果。     
基于面向对象的思想，这里的每一个圆都是一个对象，并且是继承与Animation的对象。  
那么继承Animation意味着什么呢？  
对View来说，他做动画就是拿到Animation对象中的Matrix对象，然后进行正交。在这个例子中，每一个圆都要做动画，但是，他们做动画的时间不同，动画的参数也不一样，所以，可以把他们理解为不同的动画对象。


然后，在考虑怎样**定义坐标**，以构成一个特定的几何形状。
![](http://i.imgur.com/aEfhTsH.png)  
这里正六边形的每一个顶点坐标都是相对对坐标，后面会在View的Canvas坐标体系中，将这个正六边形绘制出来。对于相对坐标，你可以进行等比例的缩放，而不会影响他们的相对位置。

#### 下拉时的移动

在下拉的时候，所有的圆都会做相同的动画，如平移、alpha值的改变，但是每一个圆的动画的值并不相同，这些值与每一个圆的圆心位置有关。
在下拉前，每一个圆都有一个初始位置，当下拉到一定距离后，他们会平移到最终要做动画的位置，所以，这其实是一个找到动画起点、终点，并根据下拉距离计算出当前动画位置的过程。

#### 刷新时的动画

这个动画的思路是每隔一段时间post一个runnable任务，在run方法中开启新的动画。
  
**Animation.start()的困惑**
  
Animation.start()方法并不能让你真正的开启动画，它只是设置了动画参数，只有当Animation.getTransformation()方法被调用，applyTransformation()方法被进行有效重写,所谓的动画才真正得以执行。  

参考ViewGroup的源码，在drawChild()方法中会调用Animation.getTransformation()方法，如果返回true，证明动画还没有结束，则取出Transformation中的Matrix与Canvas进行正交运算。再参看Animation的子类，TranslateAnimation，他重写了applyAnimation()方法，即改变Matrix。   
   
在onDraw()中调用getTransformation()的意义在于,将当前绘制时间传进去，以便Animation根据设置的参数判断动画是否已经结束，并且，不同的Animation实现类可以重写applyTransformation()方法，改变要做正交的Matrix对象，从而实现不同的动画效果。
  
明白了动画的原理，此时我们有两种思路：  
两者都要重写applyTransformation()方法

1. 通过改变Matrix来实现动画效果
2. 不改变Matrix，仅仅拿到时间插值，改变绘制时的参数(如圆心坐标)

无论采用哪种方法，绘制圆的任务都要交给"圆对象"，也就是之前说的继承自Animation的对象。它要画圆，则必须拿到Canvas,这里它拿到的是View的Canvas。


#### 具体代码分析

**静态布局**

在你动手下拉之前已经做了很多工作，比如，确定了每一个圆的起始位置和结束位置
确定了几何图形最终显示的位置，在View中的位置(居中、上下偏移量)。

1. 初始化
   将所有的圆对象创建出来
2. 测量
   计算出几何图形在View上显示的位置，确定偏移量，计算出所有圆的圆心坐标

**下拉操作**

1. 重置圆的初始位置  
   下拉前的准备工作，计算出每一个圆做动画的起始位置。
2. 为每一个圆计算出offsetX,offsetY
   考虑参数：圆的圆心坐标
3. 矩阵正交，将正交后的画布交给"圆对象"

**加载操作**

1. 开启Runnable任务，主要是配置动画参数
2. 开始动画（前提是Animation.start()方法已经执行），执行Animation.getTransformation()方法  
3. getTransformation方法会调用applyTransformation方法，在该方法中拿到时间插值，计算出圆心坐标
4. 将画布传递给"圆对象"

再次说明一下这里对Canvas的操作

* canvas.save()  
  在onDraw(Canvas canvas)中传入了一块画布，这是父组件分配的，如果我想在View上绘制出自定义的图形，一般选择的方案是先执行`canvas.save()`,即将父组件传过来的canvas对象的状态保存起来，下面进行的操作不会影响到该canvas的状态，可以理解为将该canvas压栈了。
* canvas.restore()  
  我拿到当前画布，执行translate,scale等操作，然后调用`canvas.restore()`方法，则，刚才压栈的canvas被弹出，我刚才所做的操作并没有对该canvas对象造成影响。

那么，这两种方法的意义究竟是什么呢？
  
以这个例子来说明吧。

在onDraw()方法中，使用for循环将集合中的"圆对象"一一画出来，那么，每一个"圆对象"都要拿到一个canvas对象。如果圆A拿到的canvas平移了(50,50)，那么接下来的圆B是不该拿到平移过的canvas,他应该拿到相对的没有经过任何正交的canvas，但是，onDraw()中传入却是同一个canvas对象。那么，该问题的解决办法就是用`canvas.save()`和`canvas.restore()`方法，每一个"圆对象"拿到的画布互不影响。  
  
**圆对象**

我们将屏幕上看到的一个个圆抽象成一个个"圆对象"，这些"圆对象"负责将圆画出来，所以，他需要一块画布(canvas)。

同时呢，这些圆还可以在屏幕上做动画，也就是说，这些"圆对象"也要具备"动"的能力，所以，让他继承自Animation，那么，他就成了一个可以动的对象。

所有的动画都可以通过重写`applyTransformation()`方法来实现，对于这个简单的位移动画，这里只需要简单的改变圆心的坐标就可以了。

**动画的结束位置**

在刷新动画中，起始位置很简单，就是圆对象的圆心坐标，那么，终止位置怎么算出来呢？

且看此图

![](http://i.imgur.com/fIvoRtI.png)

start和end之间的距离： l  
end和center之前的距离: 1.5l

假设start和end之间的距离是l,我们以这个距离作为参照距离，我们希望center和end之间的距离是1.5l,center和start的坐标都是知道的，那么end的坐标肯定是可以求出来的。

	 public static Point getEndPoint(Point start, Point end, float scale) {
        double start_end_dis = Math.sqrt(Math.pow((start.x - end.x), 2) + Math.pow((start.y - end.y), 2));
        double scaled_dis = start_end_dis * scale;
        int x;
        int y;
        //算出斜率
        float slop_y = (float) (1.0f * (end.y - start.y) / start_end_dis);
        y = (int) (scaled_dis * slop_y + start.y);

        float slop_x = (float) (1.0f * (end.x - start.x) / start_end_dis);
        x = (int) (scaled_dis * slop_x + start.x);

        return new Point(x, y);
    }
