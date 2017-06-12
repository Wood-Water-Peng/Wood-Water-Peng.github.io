---
layout: post
title: "uCrop开源项目分析——核心业务"
excerpt: "uCrop开源项目分析"
tags: 
- bitmap
- matrix
- Android
categories:
- Android
comments: true
share: true
---


### 本文主要是对uCrop开源项目的核心业务逻辑分析

<a href="https://github.com/Yalantis/uCrop">Github地址</a>  
<a href="http://www.pengjiebest.com/articles/uCrop-translation/">中文说明</a>

<!-- more --> 

----

#### 图片是怎么显示出来

* 从本地相册中获取
* 从网络加载

##### 从本地相册中获取
1. 细节处理
  
	* 如果没有读取SD卡的权限，则请求获取该权限
	* 有权限则调用startActivityForResult()方法
 
2. SimpleActivity拿到图片的uri,启动CropActivity
3. CropActivity拿到图片的uri之后，开启异步任务解析出bitmap返回
	* 如何将原图片的宽高解析为裁剪区的宽高
	
		在解析图片之间，已经将剪切区绘制出来，所以剪切区的宽高(requireWidth、requireHeight)是确定的，在解析的时候将该宽高带进去，作为图片的显示宽高（必须对Drawable进行再次封装，重写getInstrinsicWidth()和getInstrinsicHeight()方法）
       
	* 如何将解析出来的bitmap返回
		在调用解析方法的时候，创建出回调接口，在回调方法中拿到bitmap

#### 响应用户的手势
经过简单的逻辑分析，至此，图片已经完全显示出来，且宽高和剪切区宽高完全一致。

**重写ImageView的setImageMatrix(Matrix matrix)方法**
  
  每当我们做出translate、rotate、scale动作，本质上都是对该矩阵进行操作，然后重绘
 
**更新图片的顶点坐标**  

  调用Matrix的mapPoints()方法

		/**
		 * Apply this matrix to the array of 2D points specified by src, and write
		 * the transformed points into the array of points specified by dst. The
		 * two arrays represent their "points" as pairs of floats [x, y].
		 * 
		 * 对src数组中的值使用当前matrix进行映射，然后写入dst数组中
		 *
		 * @param dst   The array of dst points (x,y pairs)
		 * @param src   The array of src points (x,y pairs)
		 */
		public void mapPoints(float[] dst, float[] src) {

			if (dst.length != src.length) {
				throw new ArrayIndexOutOfBoundsException();
			}
			mapPoints(dst, 0, src, 0, dst.length >> 1);
		}


理解了Matrix和ImageView的关系，就理解了整个原理。

#### 释放手指之后的回滚
调用`setImageToWrapCropBounds()`方法

**判断剪切区是否完全被图片填充**

		isImageWrapCropBounds(float[] imageCorners){
			mTempMatrix.reset();
	        mTempMatrix.setRotate(-getCurrentAngle()); //将临时矩阵逆时针旋转
			
	        float[] unrotatedImageCorners = Arrays.copyOf(imageCorners, imageCorners.length);
			//经过映射后，unrotatedImageCorners保存的是图片逆时针旋转后的顶点坐标
	        mTempMatrix.mapPoints(unrotatedImageCorners); 
			
	        float[] unrotatedCropBoundsCorners = RectUtils.getCornersFromRect(mCropRect);
			//经过映射后，unrotatedCropBoundsCorners保存的是剪切区逆时针旋转后的顶点坐标
	        mTempMatrix.mapPoints(unrotatedCropBoundsCorners);
			//将顶点坐标转化成Rect，调用其contains()方法
	        return RectUtils.trapToRect(unrotatedImageCorners).contains(RectUtils.trapToRect(unrotatedCropBoundsCorners));

		}

**用临时矩阵进行平移操作，再次判断剪切区是否被完全填充**

		//拿到图片的中心点坐标
		 float currentX = mCurrentImageCenter[0];
         float currentY = mCurrentImageCenter[1];
		//算出图片和剪切区中心点之间的距离
		 float deltaX = mCropRect.centerX() - currentX;
         float deltaY = mCropRect.centerY() - currentY;
		//临时矩阵
		 mTempMatrix.reset();
         mTempMatrix.setTranslate(deltaX, deltaY);
		//采用临时矩阵，对图片的中心坐标进行平移操作
         final float[] tempCurrentImageCorners = Arrays.copyOf(mCurrentImageCorners, mCurrentImageCorners.length);
         mTempMatrix.mapPoints(tempCurrentImageCorners);
		//剪切区是否被完全填充
         boolean willImageWrapCropBoundsAfterTranslate = isImageWrapCropBounds(tempCurrentImageCorners);
			
**用临时矩阵进行缩放操作**

   如果经过了平移操作，还不能满足剪切区被完全填充的条件，那么就必须进行缩放操作。

   ![](http://i.imgur.com/I5QNMDu.jpg)


   如上图所示，图片经过平移后，中心点已经和剪切框中心点已经重合，但是图片之前被旋转过，所以，在中心点重合之后，必须加以缩放操作。
		![](http://i.imgur.com/88s9udy.png)
			
  很重要的一点就是怎样计算出图片将要缩放的比例。同样是考验数学基本功的问题，我这里进行了画图分析。注意Rect类的mLeft表示的是该矩阵左边界的坐标，其他类似。

**开启Runnable任务**
   
   至此，我们已经得到了所需要的参数，平移的距离，缩放的比例。剩下的就是一个动画操作了，根据插值器计算出新的坐标和比例，调用图片的post和scale方法，直到时间溢出或者满足剪切区的填充条件，动画停止。

#### 裁剪操作
**理解裁剪操作的原理**

在原图片的基础上，裁剪出一块剪切区大小的矩形，这究竟是怎么实现的呢？
将bitmap想像成一个矩形，现在，我们要从这个矩形上裁出另一块矩形，Bitmap类提供了一个方法

	/**
	 * x: 在原bitmap中x轴的起始像素
	 * y: 在原bitmap中y轴的起始像素
	 * width:每一行的像素值
	 * height:每一列的像素值
	 */
     public static Bitmap createBitmap(Bitmap source, int x, int y, int width, int height) {
        //
    }

* 重新思考mCurrentImageMatrix的意义
在梳理裁剪逻辑的时候，遇到一个不能理解的地方，所以又回头看了关于图片显示的代码，终于明白了。

  1. 原始图片
  原始图片可以是任意尺寸的，但是当它在View上最初显示时，则是根据剪切区的大小显示出来的，这说明在显示之前就要对原始图片进行平移、缩放操作。
  2. 初始时显示的图片
  根据源图片和剪切区的尺寸，计算出需要平移的距离、缩放的倍数

所以，mCurrentImageMatrix保存的是相对于原图片的值，而不是相对你看到的初始图片。
再看剪切的最后一个步骤，找出剪切矩形的代码

	int top = (int) ((mCropRect.top - currentImageRect.top) / currentScale);
	int left = (int) ((mCropRect.left - currentImageRect.left) / currentScale);
	int width = (int) (mCropRect.width() / currentScale);
	int height = (int) (mCropRect.height() / currentScale);
	Bitmap croppedBitmap = Bitmap.createBitmap(viewBitmap, left, top, width, height);

在此之前，我已经对原图片进行的旋转、缩放操作，最后一步是根据剪切区大小计算出该剪切区映射到源图片的剪切区。

问题，在计算width和height的时候，为什么要除以currentScale呢？
现在只考虑两个状态  

 1.  源图片
 2.  你旋转、缩放过的图片(剪切之前，屏幕上看到的图片)

当你对初始图片进行缩放操作之后，图片的像素值会发生变化，如果是放大操作，则像素值会增加。
这里的currentScale表示的是源图片相对于当前显示图片的缩放比例，在大多数情况下，currentScale是一个小于1的数，也就是说即使你将显示的图片放大了，实际上，它还是比原图片小(缩放倍数到了一定值会超过源图片)。我们看到的图片只是源图片的一个缩小版，而我们在计算这个剪切区时，是相对于源图片的，所以，必须要考虑到源图片的缩放比例。同样的道理，我们看到的剪切区也只是相对于显示出来的图片，而不是相对于源图片。

在考虑这样一个场景，假如我们看到的图片的尺寸就是源图片的尺寸，此时的剪切区表示的就是裁剪的像素大小，假如你对图片执行放大操作，在计算剪切矩形的时候并不需要考虑currentScale,因为此时显示的剪切区就是相对于源图片的，无论源图片怎样缩放，显示出来的剪切区是多少像素，那么剪切的时候就剪切多少像素。

这个弯转过来后，就能理解裁剪的逻辑了。

