---
layout: post
title: "StoreHouse动画分析"
excerpt: "开源动画分析"
tags: 
- animation
- Android
categories:
- Android
comments: true
share: true
---


**说明：这篇文章可以看做是上一篇<a href="http://www.pengjiebest.com/articles/Ptr-Refresh-Analysis/">Pull-to-refresh分析</a>的延续，对其中的一个head进行分析，拓展一下关于动画的思路**

<!-- more --> 

### 主要模块
* 动画思路分析
* ViewGroup的draw流程简单分析
* 具体实现细节分析


#### 动画思路分析

看图可知，这种复杂的动画效果肯定是在view中画(draw)出来的，那么第一眼看到它时就很好奇，他是怎么把一个个字母画出来的，我的第一反应是这其中肯定有类似硬编码的过程，把每一个字母、数字进行了编码。

	 new float[]{
                        // A
                        24, 0, 1, 22,   //一条直线
                        1, 22, 1, 72,
                        24, 0, 47, 22,
                        47, 22, 47, 72,
                        1, 48, 47, 48
                },

我在坐标系上把这些数字一一标准出来，果然是字母A。   

![](http://i.imgur.com/cH15oF4.png)   
我可以做出这样一个猜想，这个头部暴露给我们一个接口，如，我们调用head.init("ABC"),那么，剩下的事情不用我们管，在下拉的时候自然会显示出我们想要的结果。再仔细观察动画效果，很明显，这么细致的动画并不是用在一个单独的字母身上，而是字母所包含的线条身上，也就是，线条才是我们最终的操作对象。   
再假设我们有这样一个函数，ArrayList<float[]> getPath(String str)，我们输入需要显示的字符，如"ABC",那么就得到一个该字符串被拆分后的float[]集合，每个float[]数组中包含四个数字，分别代表一段线条的startX、startY、endX、endY。有了line的集合，那么剩下的只需要在绘制view的时候对其进行scale、rotate、tranlate操作，即可到达我们看到的动画效果了。

#### ViewGroup的draw流程简单分析

**View动画的基本原理**

你给View设置了一个Animation，那么View在draw的过程中会拿到这个Animation，进而拿到其中的Transformation对象，再拿到其中的Matrix对象，调用Canvas的concat(matrix)方法，将子孩子的画布中的矩阵和动画矩阵做一个正交运算，然后View在绘制的时候就会将动画效果显示出来，所以，View动画的本质就是，**每一次绘制的时候都要和Transformation中的Matrix做一次正交运算**。


	/**
     * Draw one child of this View Group. This method is responsible for getting
     * the canvas in the right state. This includes clipping, translating so
     * that the child's scrolled origin is at 0, 0, and applying any animation
     * transformations.
	 * 画出ViewGroup的一个孩子。这个方法的主要作用是帮助子孩子拿到正确的canvas,包括裁切、平移操作 ，使得子孩子滚动之后的起始坐标为（0,0），同时在画布上应用动画
	 * 
     */
    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
				//......
				Transformation transformToApply = null;
        		final Animation a = child.getAnimation();
                //......
				canvas.concat(transformToApply.getMatrix());
	}



#### 具体实现细节分析
走到这一步，我们已经了解了原理，拿到了将要做动画的对象，剩下的问题就是**how**---即怎样来做这个比较复杂的动画。  
一句话概括就是，**在用户下拉的过程中，根据用户下拉的距离来对集合中的每一根line做出平移、旋转、alpha值的综合动画**。
 
>**几点说明**  
canvas.save()  
可以理解为将当前canvas的状态做一次保存(s1)，后面所有对canvas的操作,比如平移、缩放都不会改变s1的状态。  
canvas.restore()
回滚到上一个状态，如s1  
应用场景：
你在View中拿到了一个canvas引用（c1），你需要在上面对不同的drawable应用不同的动画效果（即对canvas进行matrix正交),比如，drawable_1做平移动画（c2），drawable_02做旋转动画(c3)，他们之间不能互相影响，所以从c1--->c2之后，立刻要c2--->c1,然后再c1--->c3，这样才能正确的显示出你想要的动画效果。

	public void onDraw(Canvas canvas){
		//...
		matrix.postRotate(360 * realProgress);
    	matrix.postScale(realProgress, realProgress);
    	matrix.postTranslate(offsetX, offsetY);
		//...
    }

在onDraw()函数中的核心，就是计算出每一条line的realProcess,offsetX,offsetY,然后把它画出来。把这个细节整理下来后可以套用到很多动画效果中，明白了原理，不同的动画效果仅仅是不同的数学公式而已了。

* **确定字符的最终显示位置**  
	每一条直线都有相对的其实坐标，我们队一个字母所包含的所有直线的坐标做同加同减操作，是不会影响字符之前的相对位置的。根据这个道理，我们将字符的坐标搬移到View的画布坐标中来。在平面坐标系中，可以凭借(x,y)确定一个东西的坐标,那么对于直线，可以用中点来确定直线的坐标。
			
		
		//每一条直线的偏移量
		
		float offsetX = mOffsetX + storeHouseBarItem.midPoint.x;
		
		float offsetY = mOffsetY + storeHouseBarItem.midPoint.y;

* **确定mOffsetX和mOffsetY**  
	假设我们希望将字符放在View的正中心显示，应该怎么做呢？
![](http://i.imgur.com/gnC1LMJ.png)  
 虚线框表示的是所有字符占据的区域  
 mDrawZoneWidth：虚线区域宽度  
 mDrawZoneHeight：虚线区域高度  

在对头部进行测量的时候，我们可以将offSetX和offsetY确定下来，前提是我们必须已经知道mDrawZoneWidth和mDrawZoneHeight，那么  

**怎样确定mDrawZoneWidth和mDrawZoneHeight**  

    drawWidth = Math.max(drawWidth, startPoint.x);
    drawWidth = Math.max(drawWidth, endPoint.x);
    drawHeight = Math.max(drawHeight, startPoint.y);
    drawHeight = Math.max(drawHeight, endPoint.y); 

在对字符串进行拆分的时候，对组成字符串的所有line进行遍历，找出最右边的那么点作为drawWidth,找到最上边的点作为drawHeight。

* **确定字符的初始位置**  
  做动画，其实就是从初始位置到结束位置，中间根据一组数学公司，不断的重绘即可。
  * 确定X轴上的起始值  ---采用随机数的方式
  * 确定Y轴上的起始值  --- 0  
   
* **确定进度公式**  
  在滑动的时候，我们可以拿到头部滑动的距离，但是，根据显示效果，最左边的字母最先进入，最后出去，最右边的字母最后进入，最先出去。这完全可以使用数学公式实现，不再详述。

* **发光效果**  
上面滑动产生动画的原因是，在头部的回调函数中(onUIPositionChange)，调用了invalidate函数，导致头部重绘。发光效果是在加载中产生的，回调接口为onUIRefreshBegin,此时我拿不到对应的Canvas,那我应该如何进行重绘呢？  
由于在这里自定义了一个Animation，那么，很有必要先看看一个**正常的Animation的执行流程**。

	* 你给View设置了一个Animation对象，animation
	* 你给animation设置了监听器，设置了启动时间(animation.start())
	* 做完这些，在View进行显示的时候，动画效果就出来了
	* 可是，动画究竟是怎么显示的呢?因为你点击start()的方法的源码就知道，它仅仅是设置了一个时间
	
还是要从ViewGroup的drawChild()方法开始分析：

	final Animation a = child.getAnimation();
	if (mChildTransformation == null) {
        mChildTransformation = new Transformation(); //客户端不需要设置该对象
    }
	/**
	 * 该方法会根据你之前设置的startTime,duration进行计算得到一个动画流逝时间
	 * 然后进行监听器回调，并调用applyTransformation()方法，不同的Animation重写该方法
	 * 根据时间流逝来改变animation中tranformation对象中的matrix的状态，从而在下一次绘制时，拿
	 * 到这个矩阵和canvas中的矩阵进行正交运算，绘制出动画效果
	 */
	more = a.getTransformation(drawingTime, mChildTransformation); 
	
	if（more）{           //如果动画还没有结束
		//invoke invalidate
	}

实现一个Animation的思路：  

 * 动画的目标 ---    这里是改变alpha值，不需要改变matrix
 * 动画开始---   在onUIRefreshBegin()中改变状态，发起重绘请求，开启一个runnable任务，在onDraw中会根据状开启动画，即调用animation.getTransformation(必须手动调用)。
 * 动画结束---   将runnable任务从消息队列中移除  

比较棘手的一个问题：  
上面我们搞清楚了why的问题，即原理性问题，问什么要这么做，现在要明白怎么做，即how的问题，也就是具体的编程，算法问题，究竟怎样写来实现发光效果？
 
                  
