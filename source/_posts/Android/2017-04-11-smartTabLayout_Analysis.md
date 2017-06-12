layout: post
title: "SmartTabLayout的架构分析"
excerpt: "详细分析SmartTabLayout的结构和设计思想."
date: 2016/4/11
tags: 
- SmartTabLayout
- Android
categories:
- Android
---

#### 几个重点问题的分析

**问题是怎么发现的？**

在开源库的基础上，对参数及关键方法做出一些改动，没有达到自己预期的效果。后来才发现是有些内容没有摸透，在研究代码的过程中，始终抱着一个疑问，那就是---作者为什么要这么做。在磕磕碰碰后，终于能理解大致的思路，在这里将重点问题做出分析。

<!-- more --> 

**1.ViewPagerListener调用的时机**

根据ViewPager的源码分析,该Listener会在两个时机被调用

1. OnPageSelected---对应的源码为dispatchOnPageSelected
2. OnScrollStateChanged---对应的源码为dispatchOnSrollStateChanged


**tab的普通滑动模式**

{% img https://raw.githubusercontent.com/ogaclejapan/SmartTabLayout/master/art/demo3.gif %}

核心点 1：确定tab滑动的距离

这里，我们假定是selectedTabWidth,也就是当前tab的宽度。

作者在代码中选择selectedTab的start点来作为参考点

	int start=Utils.getStart(selectedTab);
	x+=start-startMargin+extraOffset;
	scrollTo(x,0);

start也就是该视图的左边界，相对于其父视图的位置(视图坐标)

那么，tab每次滑动的初始点，都是相对于自己getLeft()值

问题:如果上一个tab滑动后的结束位置和当前tab的起始位置不同，那么就会出现"闪现"---在滑动结束后，突然滑动一大段距离

核心点 2： 确定strip的滑动距离

strip的本质是在父视图上画一个Rect，只要能确定出Rect的left和right就行了。

起始	left=selectedStart  
起始 right=selectedEnd

至于，在移动过程中的left和right的确定，是由不同的interpolater确定的

	   float startOffset = indicationInterpolator.getLeftEdge(selectionOffset);
       float endOffset = indicationInterpolator.getRightEdge(selectionOffset);

	   int nextStart = Utils.getStart(nextTab, indicatorWithoutPadding);
       int nextEnd = Utils.getEnd(nextTab, indicatorWithoutPadding);

	   left = (int) (startOffset * nextStart + (1.0f - startOffset) * left);
       right = (int) (endOffset * nextEnd + (1.0f - endOffset) * right);


为了确定left和right公式的正确性，只需要将0和1两个极端值带进去，得出的结果是正确的就ok了

left的确定

<table>
<thead>
<tr>
  <th>StartOffset</th>
  <th>left</th>
</tr>
</thead>
<tbody>
<tr>
  <td>0</td>
  <td>left(起点)</td>
</tr>
<tr>
  <td>1</td>
  <td>nextStart(终点)</td>
</tr>
</tbody>
</table>

right的确定

<table>
<thead>
<tr>
  <th>RightOffset</th>
  <th>right</th>
</tr>
</thead>
<tbody>
<tr>
  <td>0</td>
  <td>right(起点)</td>
</tr>
<tr>
  <td>1</td>
  <td>nextEnd(终点)</td>
</tr>
</tbody>
</table>




**TabStrip的Smart滑动**

{% img https://raw.githubusercontent.com/ogaclejapan/SmartTabLayout/master/art/demo1.gif %}

该效果是由特定的Interplator实现的，如这里的SmartTabIndicationInterpolator


	    public SmartIndicationInterpolator(float factor) {
            leftEdgeInterpolator = new AccelerateInterpolator(factor); //加速器
            rightEdgeInterpolator = new DecelerateInterpolator(factor);//减速器
        }
        @Override
        public float getLeftEdge(float offset) {
            return leftEdgeInterpolator.getInterpolation(offset);
        }
        @Override
        public float getRightEdge(float offset) {
            return rightEdgeInterpolator.getInterpolation(offset);
        }

在滑动的过程中，offset为两个ViewPager之间滑动的比例，范围为0f-1f,strip的left由加速插值器决定，right由减速插值器决定。




**Tab滑动---Title offset Auto Center**

{% img https://raw.githubusercontent.com/ogaclejapan/SmartTabLayout/master/art/demo2.gif %}

该模式的关键点在于，如何确定strip在布局中的中点，不同长度的strip对应的中点也是不同的

1.先确定中点

先这样假设，假如我们要滑动的是一个点，那么所谓的中点肯定是在`getWidth()/2`,但是，现在是一条线段，他自己也是有长度的。

	 x = Utils.getWidthWithMargin(selectedTab) / 2 - getWidth() / 2;

这段代码其实算出的是该strip在布局中左顶点坐标。

2.确定每次滑动的距离

假设这样一个场景，你此时已经滑动到了中点，那么怎样保证在接下来的滑动中，strip一直处于中间呢？
诀窍就是：strip向右滑动了多少，整个tab便向左滑动多少

	int selectHalfWidth = Utils.getWidth(selectedTab) / 2 + Utils.getMarginEnd(selectedTab);
    int nextHalfWidth = Utils.getWidth(nextTab) / 2 + Utils.getMarginStart(nextTab);
	int extraOffset = Math.round(positionOffset * (selectHalfWidth + nextHalfWidth));

extraOffset便是strip向右滑动的距离。


这句代码用来设置x的临界值，假如，屏幕上显示了4个tab，每个长度为100，假如，此时在滑动第一个tab，那么x=50-200=150,就算该tab完全滑动完，也还没到临界值，此时滑动第二个tab，此时，x还是为150,注意，对于整个tab而言，已经滑动了100(tab1的宽度)的距离，所以在第二个tab滑动到50的时候，就到达临界值，此时tab开始滑动。


**Tab滑动---Always in center**

在该模式下，第一个tab必须放在整个布局的中间位置。
