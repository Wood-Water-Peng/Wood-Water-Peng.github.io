---
layout: post
title: "Ptr开源项目分析"
excerpt: "Ptr开源项目分析"
tags: 
- Ptr-Refresh
- Android
categories:
- Android
comments: true
share: true
---



**本文是对android-Ultra-Pull-To-Refresh开源项目的分析**  

<a href="https://github.com/liaohuqiu/android-Ultra-Pull-To-Refresh">Github地址</a>

---

### 主要模块  

* 滑动原理分析
* 测量布局分析
* 事件分发分析
* 滑动过程分析

<!-- more --> 

#### 滑动原理分析  

* OffsetTopAndBottom实现  
  这是View中的方法，且不管方法中的其他细节，我们只需要知道这个方法会改变View的**mTop**和**mBottom**变量，那么这两个变量是怎么达到让控件滑动的效果呢？   
  因为这两个参数会影响到子孩子拿到画布的位置和大小
* ScrollTo实现  
  我们对父控件调用ScrollTo()方法，会改变父控件的mScrollX、mScrollY方法，那么所有子孩子拿到的画布的位置都会受到影响  
  *注：以上方法可参看Android2.3源码，《Android内核剖析》*

#### 测量布局分析  

* onMeasure()   
  1. 采用默认的方法super.onMeasure()将父布局自己测量出来
  2. 测量头部，将margin和padding加上后进行测量
  3. 测量Content，加上margin和padding
* onLayout()    
    **offset的作用：**  
	offset表示头部当前的偏移量，在布局的时候一定要加上这个参数，but why?
	因为layout是一个系统回调函数，如果你在本地做了一些UI更新的操做，可能会导致系统回调该函数
    即重新给子孩子布局，那么可能出现意想不到的结果。   
    **以本demo为例：**  
	你在下拉的时候，头部的UI状态会有变化，那么会导致系统调用onLayout()方法。那么，究竟对界面的显示有什么影响呢？ 因为这个项目的滑动是使用offsetTopAndBottom,即改变View的画布位置，该操作不会导致onLayout()函数的调用，而UI的改变，如setVisibility()函数会导致onLayout()的调用，进而导致重绘，如果在重绘的时候，他们的位置不吻合，那么界面的显示就会出错。

  1. 将头部的mTop设置为-(HeaderHeight+offset)
  2. 将头部的mBottom设置为0
  3. 将Content正常放置  
  *注：上述三个步骤需考虑到父布局的padding和子布局的margin*

#### 事件分发分析
![](http://i.imgur.com/1eNEVUE.png)

*  **关于头部添加的几种方式**
	1. xml中   
		确保所有的头部都实现了PtrUIHandler接口，因为对于PtrContainer来说，他看到的都是接口，看不到实际的对象，此所谓面向接口编程
	2. 代码动态添加  
		只需要调用setHeadView()方法即可
*  **关于子孩子个数的约定**
    1. 大于两个  抛出异常  
    2. 2个      正常  
    3. 1个      不管用户设置的是head或者content，这里都当做content  
    4. 0个      添加一个errorView作为content显示   

*  **关于Content的类型**
    1. 普通类型    
    		如RelativeLayout、LinearLayout等一般布局
    2. 可滚动类型  
	    	如ListView、ScrollView等可滚动的布局
* **需要注意的几点**
	1. DOWN事件的处理   
	   如果Header和Content都是普通类型，那么他们都不会消费down事件，那么该Container就必须返回true，才能保证能够收到后续的事件。如果Content是可滚动型的，那么大可不必担心这个问题，因为可滚动类型能够消费down事件。
  
    2. 移动过程的合理分发   
       在滑动的过程中，Container最先拿到MOVE事件，如果有必要，他会就这个事件进行移动操作，然后选择是否继续将这个事件传递下去，比如有时间我滑动，但动的是Content，那就是父类把事件传给了Content，有时候动的是所有子孩子，那么就是父类消费了该事件，而Content并没有拿到

* **状态的切换**  
    1. init--->prepare   
       头部完全隐藏的状态为init，一旦滑动，则变成prepare状态
  	2. prepare--->loading    
  	   当你滑动的距离超过了刷新距离，那么当你释放后，会进入loading状态，当距离不够时，则直接回滚到初始位置  
    3. loading--->completed  
       当加载完后，会从loading变成completed，然后回滚到init状态，需要注意，如果在loading状态，那么在滑动的时候是不能改变界面显示的（根据状态判断）

* **回滚**   
	使用ScrollChecker类来辅助执行，内部采用Scroller类来计时，然后不断post自己到UI线程的MessageQueue即可（因为该runnable内部要执行UI操作）

#### 关于Head的耦合实现

   Head一般采用动态添加的方式，所有的Head必须实现PtrUIHandler接口，用于处理滑动过程中的UI更新，Container在滑动的处理中，只管调用该接口的方法。  
   Head在回调方法中可以拿到Container的引用，进而拿到滑动过程中的参数，如curPosition，滑动的百分比，进而对Head的UI进行合理的显示

#### LoadMore的实现

* 思路  
   在该开源项目中暂时没有LoadMore的实现，我在这里说一下思路。在现实项目中，绝大多数情况下，LoadMore功能使用在ListView中，这里以ListView举例。  
   在父控件的dispatchTouchEvent方法中，对于UP事件我们可以进行一个上拉加载的判断，对于ListView，如果已经上拉到了底部，那么我们可以进行加载更多操作	

   


		private boolean isBottom() {  
        	if (mListView.getCount() > 0) {   

            if (mListView.getLastVisiblePosition() == mListView.getAdapte().getCount() - 1
                && mListView.getChildAt(mListView.getChildCount() - 1).getBottom() <= mListView.getHeight()) {
                return true;
            }
        }
        	return false;
    	} 

 

 * 一个失败的方案  
 	我可否像提供Head一样，在Container的内部封装一个Foot，并且可以让用户自定义FootView,我们只提供接口，用户可选择各种各样的FootView具体实现。  
 	我可以在Container中提供一个setFootView()接口，内部逻辑是我在Container中找到对应的ListView，然后将Foot添加进去，那么如果在用户调用setFootView()的时候，ListView还没有被填充出来呢，而我在Container中也并不知道ListView的id而无法去手动(findViewById)找到他。

* 一个暂行的方案  
    参考RefreshLayout的实现，Container提供一个接口，内部调用ListView的自带方法来判断是否滑到底部。该方法和上一个方法的区别就是，该方法提前找到Container内部的ListView，将FootView添加进去，然后在Container中提供回调接口。我们可以提供一个默认的FootView，一般情况下使用该View即可。

