layout: post
title: "uCrop开源项目翻译"
excerpt: "uCrop开源项目翻译"
tags: 
- bitmap
- matrix
- Android
categories:
- Android
comments: true
share: true
---


###The uCrop Challenge

在开始写这个开源项目之前，我会事先定义一些特性，他们很好理解：
  
*  裁剪图片
*  支持任何裁剪比例
*  支持手势缩放、平移和旋转
*  在裁剪区域内不允许出现空白区域(剪切区必须是有效图片)
*  创建一个随时可用的Activity,它可以使用Crop view。也就是说，这个开源库包含一个Activity,这个Activity包含一个Crop view和其他另外的组件

<!-- more --> 

### Crop view
考虑到我打算构造的特性，我决定将View的逻辑划为三层。

1.TransformImageView extends ImageView
  
* 接受源图片
* 将变换矩阵(translate,scale,rotate)应用到当前图片
  
这一层不关心裁剪和触摸手势

2.CropImageView extends TransformImageView

* 绘制裁剪边界和网格线
* 为裁剪边界设置图片(如果用户缩放或者旋转图片导致裁剪框内出现了空白区域，那么图片应该自动的平移/旋转来适应裁剪边界，并且没有空白区域)
* 继承方法，明确处理变换矩阵的规则(限制最小和最大的缩放比例)
* 添加缩放方法(动画变换)
* 裁剪图片

这一层几乎包含了裁剪图片的所有功能，但它只指定了做这些事的接口，下面我们要添加手势的支持，才可以调用这些方法。

3.GestureImageView extends CropImageView

* 监听手势，调用合适的方法

### TransformImageView

这是最简单的部分。
首先，我拿到一个Uri，然后将其解码为一个合适大小的bitmap。

	ParcelFileDescriptor parcelFileDescriptor = context.getContentResolver().openFileDescriptor(uri, "r");
	FileDescriptor fileDescriptor = parcelFileDescriptor.getFileDescriptor();
 
现在，我可以使用BitmapFactory来对FileDescriptor进行解码。
但是，在解析bitmap之前，我们需要知道它的大小。因为，如果图片的分辨率太高，bitmap需要重新采样。

	final BitmapFactory.Options options = new BitmapFactory.Options();

	options.inJustDecodeBounds = true;
	
	BitmapFactory.decodeFileDescriptor(fileDescriptor, null, options);
	
	options.inSampleSize = calculateInSampleSize(options, requiredWidth, requiredHeight);
	
	options.inJustDecodeBounds = false;
		
	Bitmap decodeSampledBitmap = BitmapFactory.decodeFileDescriptor(fileDescriptor, null, options);
	
	close(parcelFileDescriptor);
	
	ExifInterface exif = getExif(uri);
	
	if (exif != null) {
	
	  int exifOrientation = exif.getAttributeInt(ExifInterface.TAG_ORIENTATION, ExifInterface.ORIENTATION_NORMAL);
	
	  return rotateBitmap(decodeSampledBitmap, exifToDegrees(exifOrientation));
	
	} else {
	
	  return decodeSampledBitmap;
	
	}

关于bitmap的尺寸，我想多说几句。

**1.怎样设置一张图片需要的宽高？**

`calculateInSampleSize(options, requiredWidth, requiredHeight)`方法在计算`SampleSize`的时候基于这样一个规则---图片的宽高都不能超规定的值。
那么，你是怎么知道一张图片它需要的宽高的呢？有些开发者使用常量来进行规定，比如，有一个裁剪库使用1000px作为图片的最大尺寸。然而，Android设备如此之多，要找到一个常量来适配所有的屏幕看起来并不是最好的办法。我也可以使用view的尺寸，或者将设备的有效内存考虑进去来计算bitmap的尺寸。

在做了一点研究之后，我决定用屏幕的对角线作为图片宽高的最大像素。

**2.我是怎样将变换应用到矩阵，并将矩阵用于更新view的呢？**

为此，我创建了3个方法：
  
* position
* scale 
* rotation angle

		public void postScale(float deltaScale, float px, float py) {
 			if (deltaScale != 0) {
      			mCurrentImageMatrix.postScale(deltaScale, deltaScale, px, py);
      			setImageMatrix(mCurrentImageMatrix);
  			}
		}

并没有很神奇的地方：这些方法会检查给到的参数是否为0，然后将他应用到当前的图片矩阵中。
在这里我重写了`setImageMatrix()`方法，它会调用ImageView的`setImageMatrix()`方法，并将给到的matrix参数带过去，它同时调用`updateCurrentImagePoints()`方法，该方法会更新一些在CropImageView中需要的变量。


### CropImageView

**Crop guidelines**

在`TransformImageView`上面画出裁剪网格线。

**确保在裁剪框内没有空白区域**

用户必须能够同时对图片进行平移、旋转、缩放操作。同时，在用户释放图片的时候，在裁剪区域内不能有空白区域。那么，我是怎样做到这些的呢？有两种潜在的解决办法：

1. 将图片的变换限制在裁剪区内，如果图片处在裁剪区的边界，那么用户就不能进行缩放、旋转或者平移操作。
2. 让用户可以随意的移动图片，但是，当图片被释放后，图片的位置和尺寸需要被自动校正。

我选择第二种。
这样，我需要解决两个问题

* 我怎样检测剪切区域内是否已经被图片填充满了。
* 我怎样计算出将图片回滚到正确位置所需要的所有变换

**检测图片是否填满了剪切区**  

起初，我们有两个矩形：剪切框和图片。剪切框必须完全在图片矩形内部。在最小的情况下，他们的边界必须重合。当他们的x、y是对齐的时候，我们只需要调用Rect的contains()方法就可以进行判断。但是，在这里，图片矩形是可以任意旋转的。

![](http://i.imgur.com/0OABJ6H.jpg)
最开始，我一直困惑的是怎样检测一个旋转过的矩形是否包含一个轴对齐的矩形。在做了一些无谓的计算之后，我发现这个问题可以转变为：我怎样检测轴对称的矩形是否包含旋转过的矩形？
![](http://i.imgur.com/U9UrxYc.jpg)

现在看起来没这么难了！我只要检测出裁剪区的四个角落是否在图片矩阵的内部即可。  
`mCropRect`变量已经定义了。
我之前提到过一个`setImageMatrix(Matrix matrix)`方法。在这个方法里面，我们会调用到`updateCurrentImagePoints()`方法

	private void updateCurrentImagePoints() {
    	mCurrentImageMatrix.mapPoints(mCurrentImageCorners, mInitialImageCorners);
    	mCurrentImageMatrix.mapPoints(mCurrentImageCenter, mInitialImageCenter);
	}

每当图片矩阵被改变时，我都会更新图片的中点和顶点的坐标，所以，我可以写出一个方法来检测当前图片是否包含剪切区：
	
	   protected boolean isImageWrapCropBounds() {
		   mTempMatrix.reset();
		   mTempMatrix.setRotate(-getCurrentAngle());
		   float[] unrotatedImageCorners = Arrays.copyOf(mCurrentImageCorners, mCurrentImageCorners.length);
		   mTempMatrix.mapPoints(unrotatedImageCorners);
		   float[] unrotatedCropBoundsCorners = CropMath.getCornersFromRect(mCropRect);
		   mTempMatrix.mapPoints(unrotatedCropBoundsCorners);
		   return CropMath.trapToRect(unrotatedImageCorners).contains(CropMath.trapToRect(unrotatedCropBoundsCorners));
	   }

我使用一个临时的Matrix对象同时对剪切区个图片矩阵进行反旋转操作，然后使用RectF类的contains(RectF rect)方法来检测剪切区是否完全在图片矩形内部。

**变换图片使其包含剪切区**

首先，我计算出当前图片和剪切区图片中点之间的距离。然后，我使用一个临时矩阵对图片进行平移，检测它是否包含剪切区域。

	float oldX = mCurrentImageCenter[0];
	float oldY = mCurrentImageCenter[1];
	float deltaX = mCropRect.centerX() - oldX;
	float deltaY = mCropRect.centerY() - oldY;
	mTempMatrix.reset();
	mTempMatrix.setTranslate(deltaX, deltaY);
	float[] tempCurrentImageCorners = Arrays.copyOf(mCurrentImageCorners, mCurrentImageCorners.length);
	mTempMatrix.mapPoints(tempCurrentImageCorners);
	boolean willImageWrapCropBoundsAfterTranslate = isImageWrapCropBounds(tempCurrentImageCorners);

这个判断非常重要，因为图片如果在平移之后还不能完全包含剪切区，那么必须加以缩放变换。
![](http://i.imgur.com/MjzeojF.gif)
  
因此，我添加了计算缩放距离的代码：

		float currentScale = getCurrentScale();
		float deltaScale = 0;
		if (!willImageWrapCropBoundsAfterTranslate) {
		  RectF tempCropRect = new RectF(mCropRect);
		  mTempMatrix.reset();
		  mTempMatrix.setRotate(getCurrentAngle());
		  mTempMatrix.mapRect(tempCropRect);
		  float[] currentImageSides = RectUtils.getRectSidesFromCorners(mCurrentImageCorners);
		  deltaScale = Math.max(tempCropRect.width() / currentImageSides[0],tempCropRect.height() / currentImageSides[1]);
		  deltaScale = deltaScale * currentScale - currentScale;
		}

现在，我有了对图片进行**平移**和**缩放**的参数，接下我要用一个Runnable类来让他动起来。

	
	 @Override
	 public void run() {
	  long now = System.currentTimeMillis();
	  float currentMs = Math.min(mDurationMs, now - mStartTime);
	  float newX = CubicEasing.easeOut(currentMs, 0, mCenterDiffX, mDurationMs);
	  float newY = CubicEasing.easeOut(currentMs, 0, mCenterDiffY, mDurationMs);
	  float newScale = CubicEasing.easeInOut(currentMs, 0, mDeltaScale, mDurationMs);
	  if (currentMs < mDurationMs) {
     	 cropImageView.postTranslate(newX - (cropImageView.mCurrentImageCenter[0] - mOldX), newY - (cropImageView.mCurrentImageCenter[1] - mOldY));
      	if (!mWillBeImageInBoundsAfterTranslate) {
            cropImageView.zoomInImage(mOldScale + newScale, cropImageView.mCropRect.centerX(), cropImageView.mCropRect.centerY());
      	}
     	 if (!cropImageView.isImageWrapCropBounds()) {
          	cropImageView.post(this);
     	 }
 	  }
	}

使用`CubicEasing`类来对距离重新插值计算，使得动画更加自然
最终，我将这些值应用到图片矩阵上，当context为null、时间溢出或者图片完全包含裁剪区时，runnable停止。
![](http://i.imgur.com/0dKECHf.gif)

**裁剪图片** 

终于来到了最关键的一步。
首先，我拿到接下来计算需要的当前值。

	Bitmap viewBitmap = getViewBitmap();
	if (viewBitmap == null) {
	    return null;
	}
	cancelAllAnimations();
	setImageToWrapCropBounds(false); // without animation
	RectF currentImageRect = RectUtils.trapToRect(mCurrentImageCorners);
	if (currentImageRect.isEmpty()) {
	    return null;
	}
	float currentScale = getCurrentScale();
	float currentAngle = getCurrentAngle();

将要被裁剪的Bitmap,代表了当前图片状态的矩形，当前比例和旋转角度，对他们进行检测，然后接着执行下一步：

	if (mMaxResultImageSizeX > 0 && mMaxResultImageSizeY > 0) {
    float cropWidth = mCropRect.width() / currentScale;
    float cropHeight = mCropRect.height() / currentScale;
	    if (cropWidth > mMaxResultImageSizeX || cropHeight > mMaxResultImageSizeY) {
	      float scaleX = mMaxResultImageSizeX / cropWidth;
	      float scaleY = mMaxResultImageSizeY / cropHeight;
	      float resizeScale = Math.min(scaleX, scaleY);
	      Bitmap resizedBitmap = Bitmap.createScaledBitmap(viewBitmap,
	      (int) (viewBitmap.getWidth() * resizeScale),
	      (int) (viewBitmap.getHeight() * resizeScale), false);
	      viewBitmap.recycle();
	      viewBitmap = resizedBitmap;
	      currentScale /= resizeScale;
	    }
    }

使用上面的代码，我可以指定图片的输出宽高，如果你想为自己的剪切一张头像，那么该图片的最大的宽高也许就是500px。
在代码中，我首先检测最大输出值是否已经被指定，如果剪切后的图片的宽高值比最大值大，那么我就会重新对图片进行缩放，我使用`Bitmap.createScaledBitmap`方法,然后对原图片进行回收，然后改变`currentScale`值，保证后面的计算不会受影响。

现在，检测图片是否旋转：

	if (currentAngle != 0) {
	  	mTempMatrix.reset();
	    mTempMatrix.setRotate(currentAngle, viewBitmap.getWidth() / 2, viewBitmap.getHeight() / 2);
	    Bitmap rotatedBitmap = Bitmap.createBitmap(viewBitmap, 0, 0,viewBitmap.getWidth(),viewBitmap.getHeight(), mTempMatrix, true);
	    viewBitmap.recycle();
	    viewBitmap = rotatedBitmap;
    }

这里跟处理Bitmap的缩放操作一样，我会根据`currentAngle`值重新创建新的bitmap对象，将之前的bitmap回收(避免内存溢出)。

最后，计算出要从Bitmap上裁切的矩形的坐标。
top:在图片顶部的起始距离
left:在图片左边的起始距离

	int top = (int) ((mCropRect.top - currentImageRect.top) / currentScale);
	int left = (int) ((mCropRect.left - currentImageRect.left) / currentScale);
	int width = (int) (mCropRect.width() / currentScale);
	int height = (int) (mCropRect.height() / currentScale);
	Bitmap croppedBitmap = Bitmap.createBitmap(viewBitmap, left, top, width, height);
	

**GestureImageView**

这一层紧跟着拥有move、rotate、scale方法的TransformImageView层，这一层的任务就是快速拿到反馈，手势逻辑和支持的手势会随着库的开发而不断改进。

我们需要支持的手势：

**1.Zoom gestures**

图片必须对一些列的手势做出反应，并改变他的缩放值。

* 双击放大
* 两根手指缩小放大 

**2.Scroll gestures**

通过手指滑动图片


**3.旋转手势**

用户可以将两根手指方在图片上，做出旋转操作就可以旋转图片。
更重要的是，所有的手势必须可以同步做出响应，所有的图片变换必须根据用户手指间的焦点执行，这样你才会感觉，是你拖着图片在屏幕上移动。

Android SDK提供了两个很方便的类：GestureDetector和ScaleGestureDetector。但是，并有提供缩放手势的检测类，所以我只要自己写一个了，下面是所有的手势监听类：

		private class ScaleListener extends ScaleGestureDetector.SimpleOnScaleGestureListener
		  @Override
		  public boolean onScale(ScaleGestureDetector detector) {
		      postScale(detector.getScaleFactor(), mMidPntX, mMidPntY);
		      return true;
		  }
		}

		private class GestureListener extends GestureDetector.SimpleOnGestureListener {
		  @Override
		  public boolean onDoubleTap(MotionEvent e) {
		      zoomImageToPosition(getDoubleTapTargetScale(), e.getX(), e.getY(), DOUBLE_TAP_ZOOM_DURATION);
		      return super.onDoubleTap(e);
		  }
		  @Override
		  public boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY) {
		      postTranslate(-distanceX, -distanceY);
		      return true;
		  }
		}

		private class RotateListener extends RotationGestureDetector.SimpleOnRotationGestureListener {
		  @Override
		  public boolean onRotation(RotationGestureDetector rotationDetector) {
		      postRotate(rotationDetector.getAngle(), mMidPntX, mMidPntY);
		      return true;
		  }
	}

监听器创建出来后，只要在onTouchEvent()中将事件传递给这些监听器就行了。
现在让我们假设这样一个场景，用户将图片的一部分拖出了屏幕，然后释放了所有的手指。在那一刻，`ACTION_UP`事件触发了`setImageToCropBounds()`方法	。图片会启动动画回滚到剪切区域内，当动画在执行的时候，用户又触摸了图片，`ACTION_DOWM`事件被触发，接着动画被取消了，下面图片的变换就取决于用户的手势了。

**UCropActivity**

![](http://i.imgur.com/Sdt3Vqr.jpg)

为了完成这个库，我还需要做一些事情：
* 展示水平滑动控件的custom view
* 展示裁剪比例的custom view	
		
Activity从UCrop类中拿到所有数据，UCrop采用builder设计模式。




	
		
		
		
		
	
		
			
		