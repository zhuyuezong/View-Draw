# Android视图绘制原理
## 概述
Android系统的视图结构的设计也采用了组合模式，即View作为所有图形的基类，Viewgroup对View继承扩展为视图容器类，由此就得到了视图部分的基本结构--树形结构<br>
![image](https://github.com/zhuyuezong/View-Draw/blob/master/images/1343827041_4739.png?raw=true)<br>
整个绘制流程从ViewRootImp类的performtraversals()方法开始，函数的主要功能是判断是否要重新计算measure，是否要重新布局layou，是否要重新绘制draw。流程图如下：<br><br>
![iamge](https://github.com/zhuyuezong/View-Draw/blob/master/images/View的绘制流程.jpg?raw=true)
## 递归调用measure
整个view从RootView开始递归调用measure方法，View的measure方法是final类型，是不能被子类重写的，在measure中调用onMeasure方法。子类可以重写onMeasure方法。如果是View，调用onMeasure就完成了测量工作。如果是ViewGroup就需要递归去调用子类View的measure的方法，依次递归。
```Java
//final方法，子类不可重写
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
      ......
      //回调onMeasure()方法
      onMeasure(widthMeasureSpec, heightMeasureSpec);
      ......
}
```
方法中的两个参数由父类传过来的，每个参数由两部分组成，高2位表示MODE，定义在MeasureSpec类（View的内部类）中，有三种类型，MeasureSpec.EXACTLY表示确定大小， MeasureSpec.AT_MOST表示最大大小， MeasureSpec.UNSPECIFIED不确定。低30位表示size，也就是父View的大小。对于系统Window类的DecorVIew对象Mode一般都为MeasureSpec.EXACTLY ，而size分别对应屏幕宽高。对于子View来说大小是由父View和子View共同决定的。<br>
我们再来看看onMeasure方法
```Java
/**
 * <p>
 * Measure the view and its content to determine the measured width and the
 * measured height. This method is invoked by {@link #measure(int, int)} and
 * should be overriden by subclasses to provide accurate and efficient
 * measurement of their contents.
 * </p>
 *
 * <p>
 * <strong>CONTRACT:</strong> When overriding this method, you
 * <em>must</em> call {@link #setMeasuredDimension(int, int)} to store the
 * measured width and height of this view. Failure to do so will trigger an
 * <code>IllegalStateException</code>, thrown by
 * {@link #measure(int, int)}. Calling the superclass'
 * {@link #onMeasure(int, int)} is a valid use.
 * </p>
 *
 * <p>
 * The base class implementation of measure defaults to the background size,
 * unless a larger size is allowed by the MeasureSpec. Subclasses should
 * override {@link #onMeasure(int, int)} to provide better measurements of
 * their content.
 * </p>
 *
 * <p>
 * If this method is overridden, it is the subclass's responsibility to make
 * sure the measured height and width are at least the view's minimum height
 * and width ({@link #getSuggestedMinimumHeight()} and
 * {@link #getSuggestedMinimumWidth()}).
 * </p>
 *
 * @param widthMeasureSpec horizontal space requirements as imposed by the parent.
 *                         The requirements are encoded with
 *                         {@link android.view.View.MeasureSpec}.
 * @param heightMeasureSpec vertical space requirements as imposed by the parent.
 *                         The requirements are encoded with
 *                         {@link android.view.View.MeasureSpec}.
 *
 * @see #getMeasuredWidth()
 * @see #getMeasuredHeight()
 * @see #setMeasuredDimension(int, int)
 * @see #getSuggestedMinimumHeight()
 * @see #getSuggestedMinimumWidth()
 * @see android.view.View.MeasureSpec#getMode(int)
 * @see android.view.View.MeasureSpec#getSize(int)
 */
 //View的onMeasure默认实现方法
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```
从注释可以看到在这个方法里一定要调用setMeasuredDimension这个方法来设置View的高宽。View默认的高框有背景决定。

```Java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```
从上面源码可以看出，从父类传的宽高高位的mode和低位的size。如果是AT_MOST或者EXACTLY就返回specSize.<br>
继续看上面onMeasure方法。getSuggestedMinimumWidth与getSuggestedMinimumHeight都是View的方法，具体如下：
```Java
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}

protected int getSuggestedMinimumHeight() {
    return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());
}
```
可以看出是由View的背景和mMinHeight的值决定，这个参数可以直接在布局文件中定义。<br>
到这里，View的测量也就算完成了。然后ViewGroup是需要递归测量ChildView的大小，所以在ViewGroup里定义了相关递归测量的方法，比如 measureChild和 measureChildWithMargins，区别在于是否把margin也作为子视图的大小。
```Java
/**
 * Ask one of the children of this view to measure itself, taking into
 * account both the MeasureSpec requirements for this view and its padding
 * and margins. The child must have MarginLayoutParams The heavy lifting is
 * done in getChildMeasureSpec.
 *
 * @param child The child to measure
 * @param parentWidthMeasureSpec The width requirements for this view
 * @param widthUsed Extra space that has been used up by the parent
 *        horizontally (possibly by other children of the parent)
 * @param parentHeightMeasureSpec The height requirements for this view
 * @param heightUsed Extra space that has been used up by the parent
 *        vertically (possibly by other children of the parent)
 */
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
measureChild与之的区别就在于计算specSize的时候没有加上margin值。最后会调用child.measure，就回到了我们上面说的问题。<br>
ViewGroup并没有实现这两个方法的调用，这些方法的调用在其子类，如RelativeLayout，LinearLayout类的onMeasure方法中对child遍历调用，这里注意一点，调用方法在setMeasuredDimension方法之前，也就是说先测量完成子布局w再决定父布局的大小。<br>
MeasureSpec.EXACTLY //确定模式，父View希望子View的大小是确定的，由specSize决定；<br>
MeasureSpec.AT_MOST //最多模式，父View希望子View的大小最多是specSize指定的值；<br>
MeasureSpec.UNSPECIFIED //未指定模式，父View完全依据子View的设计值来决定；<br>

## 递归调用layout
在ViewRootImpl类的performtraversals方法中，执行完measure之后，就会执行layout方法
```Java
private void performTraversals() {
    ......
    mView.layout(0, 0, mView.getMeasuredWidth(), mView.getMeasuredHeight());
    ......
}
```
可以看到，layout方法的参数包括view的上下左右的坐标，并且左上都为0。<br>
```Java
/**
  * Assign a size and position to a view and all of its
  * descendants
  *
  * <p>This is the second phase of the layout mechanism.
  * (The first is measuring). In this phase, each parent calls
  * layout on all of its children to position them.
  * This is typically done using the child measurements
  * that were stored in the measure pass().</p>
  *
  * <p>Derived classes should not override this method.
  * Derived classes with children should override
  * onLayout. In that method, they should
  * call layout on each of their children.</p>
  *
  * @param l Left position, relative to parent
  * @param t Top position, relative to parent
  * @param r Right position, relative to parent
  * @param b Bottom position, relative to parent
  */
 @SuppressWarnings({"unchecked"})
 public void layout(int l, int t, int r, int b) {
     if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
         onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
         mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
     }

     int oldL = mLeft;
     int oldT = mTop;
     int oldB = mBottom;
     int oldR = mRight;

     boolean changed = isLayoutModeOptical(mParent) ?
             setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

     if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
         onLayout(changed, l, t, r, b);
         mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

         ListenerInfo li = mListenerInfo;
         if (li != null && li.mOnLayoutChangeListeners != null) {
             ArrayList<OnLayoutChangeListener> listenersCopy =
                     (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
             int numListeners = listenersCopy.size();
             for (int i = 0; i < numListeners; ++i) {
                 listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
             }
         }
     }

     mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
     mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
 }
```
从源码可以找到，在layout方法里面，先执行setFrame把参数分别赋值给mLeft、mTop、mRight和mBottom这几个变量,然后判断是否需要重新layout，需要则执行onLayout方法，这里需要注意的是View的onLayout方法是一个空方法。而ViewGroup中onLayout方法是一个抽象方法，也就是说它的子类一定要重写onLayout方法。比如LinearLayout.
```Java
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    if (mOrientation == VERTICAL) {
        layoutVertical(l, t, r, b);
    } else {
        layoutHorizontal(l, t, r, b);
    }
}
```
根据不同布局在看看layoutVertical,在layoutVertical中递归所有子View并调用setChildFrame，在setChindFrame方法中
```Java
private void setChildFrame(View child, int left, int top, int width, int height) {        
   child.layout(left, top, left + width, top + height);
}
```
这样就开始递归调用。所以我们知道，layout的主要作用是标记view的上下左右的位置，为后面的绘制做准备。

## 递归调用draw

在上面的measure和layout调用完以后，就真正的将这些视图绘制在屏幕上，直接上源码
```Java
public void draw(Canvas canvas) {
       final int privateFlags = mPrivateFlags;
       final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
               (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
       mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

       /*
        * Draw traversal performs several drawing steps which must be executed
        * in the appropriate order:
        *
        *      1. Draw the background
        *      2. If necessary, save the canvas' layers to prepare for fading
        *      3. Draw view's content
        *      4. Draw children
        *      5. If necessary, draw the fading edges and restore layers
        *      6. Draw decorations (scrollbars for instance)
        */

       // Step 1, draw the background, if needed
       int saveCount;

       if (!dirtyOpaque) {
           drawBackground(canvas);
       }

       // skip step 2 & 5 if possible (common case)
       final int viewFlags = mViewFlags;
       boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
       boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
       if (!verticalEdges && !horizontalEdges) {
           // Step 3, draw the content
           if (!dirtyOpaque) onDraw(canvas);

           // Step 4, draw the children
           dispatchDraw(canvas);

           // Overlay is part of the content and draws beneath Foreground
           if (mOverlay != null && !mOverlay.isEmpty()) {
               mOverlay.getOverlayView().dispatchDraw(canvas);
           }

           // Step 6, draw decorations (foreground, scrollbars)
           onDrawForeground(canvas);

           // we're done...
           return;
       }

       /*
        * Here we do the full fledged routine...
        * (this is an uncommon case where speed matters less,
        * this is why we repeat some of the tests that have been
        * done above)
        */

       boolean drawTop = false;
       boolean drawBottom = false;
       boolean drawLeft = false;
       boolean drawRight = false;

       float topFadeStrength = 0.0f;
       float bottomFadeStrength = 0.0f;
       float leftFadeStrength = 0.0f;
       float rightFadeStrength = 0.0f;

       // Step 2, save the canvas' layers
       int paddingLeft = mPaddingLeft;

       final boolean offsetRequired = isPaddingOffsetRequired();
       if (offsetRequired) {
           paddingLeft += getLeftPaddingOffset();
       }

       int left = mScrollX + paddingLeft;
       int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
       int top = mScrollY + getFadeTop(offsetRequired);
       int bottom = top + getFadeHeight(offsetRequired);

       if (offsetRequired) {
           right += getRightPaddingOffset();
           bottom += getBottomPaddingOffset();
       }

       final ScrollabilityCache scrollabilityCache = mScrollCache;
       final float fadeHeight = scrollabilityCache.fadingEdgeLength;
       int length = (int) fadeHeight;

       // clip the fade length if top and bottom fades overlap
       // overlapping fades produce odd-looking artifacts
       if (verticalEdges && (top + length > bottom - length)) {
           length = (bottom - top) / 2;
       }

       // also clip horizontal fades if necessary
       if (horizontalEdges && (left + length > right - length)) {
           length = (right - left) / 2;
       }

       if (verticalEdges) {
           topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
           drawTop = topFadeStrength * fadeHeight > 1.0f;
           bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
           drawBottom = bottomFadeStrength * fadeHeight > 1.0f;
       }

       if (horizontalEdges) {
           leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
           drawLeft = leftFadeStrength * fadeHeight > 1.0f;
           rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
           drawRight = rightFadeStrength * fadeHeight > 1.0f;
       }

       saveCount = canvas.getSaveCount();

       int solidColor = getSolidColor();
       if (solidColor == 0) {
           final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;

           if (drawTop) {
               canvas.saveLayer(left, top, right, top + length, null, flags);
           }

           if (drawBottom) {
               canvas.saveLayer(left, bottom - length, right, bottom, null, flags);
           }

           if (drawLeft) {
               canvas.saveLayer(left, top, left + length, bottom, null, flags);
           }

           if (drawRight) {
               canvas.saveLayer(right - length, top, right, bottom, null, flags);
           }
       } else {
           scrollabilityCache.setFadeColor(solidColor);
       }

       // Step 3, draw the content
       if (!dirtyOpaque) onDraw(canvas);

       // Step 4, draw the children
       dispatchDraw(canvas);

       // Step 5, draw the fade effect and restore layers
       final Paint p = scrollabilityCache.paint;
       final Matrix matrix = scrollabilityCache.matrix;
       final Shader fade = scrollabilityCache.shader;

       if (drawTop) {
           matrix.setScale(1, fadeHeight * topFadeStrength);
           matrix.postTranslate(left, top);
           fade.setLocalMatrix(matrix);
           p.setShader(fade);
           canvas.drawRect(left, top, right, top + length, p);
       }

       if (drawBottom) {
           matrix.setScale(1, fadeHeight * bottomFadeStrength);
           matrix.postRotate(180);
           matrix.postTranslate(left, bottom);
           fade.setLocalMatrix(matrix);
           p.setShader(fade);
           canvas.drawRect(left, bottom - length, right, bottom, p);
       }

       if (drawLeft) {
           matrix.setScale(1, fadeHeight * leftFadeStrength);
           matrix.postRotate(-90);
           matrix.postTranslate(left, top);
           fade.setLocalMatrix(matrix);
           p.setShader(fade);
           canvas.drawRect(left, top, left + length, bottom, p);
       }

       if (drawRight) {
           matrix.setScale(1, fadeHeight * rightFadeStrength);
           matrix.postRotate(90);
           matrix.postTranslate(right, top);
           fade.setLocalMatrix(matrix);
           p.setShader(fade);
           canvas.drawRect(right - length, top, right, bottom, p);
       }

       canvas.restoreToCount(saveCount);

       // Overlay is part of the content and draws beneath Foreground
       if (mOverlay != null && !mOverlay.isEmpty()) {
           mOverlay.getOverlayView().dispatchDraw(canvas);
       }

       // Step 6, draw decorations (foreground, scrollbars)
       onDrawForeground(canvas);
   }
```
看到注释中：
// Step 1, draw the background, if needed 绘制背景<br>
// Step 2, save the canvas' layers 保存绘制图层<br>
// Step 3, draw the content 绘制内容 比如TextView绘制文字<br>
// Step 4, draw the children 绘制子类，这个方法是给ViewGroup使用的<br>
// Step 5, draw the fade effect and restore layers 绘制淡入淡出，并恢复图层 <br>
// Step 6, draw decorations (foreground, scrollbars) 绘制顶端视图，比如scroll bars <br>

## View的invalidate和postInvalidate
invalidate方法通过invalidateChild不断向上级View回溯，将需要刷新的区域回调，直到ViewRootImpl的invalidateChildInParent方法
```Java
public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
   ......
   //View调运invalidate最终层层上传到ViewRootImpl后最终触发了该方法
   scheduleTraversals();
   ......
   return null;
}
```
scheduleTraversals会通过Handler的Runnable发送一个异步消息，调运doTraversal方法，然后最终调用performTraversals()执行重绘。

## View的requestLayout方法
直接上代码
```Java
public void requestLayout() {
    ......
    if (mParent != null && !mParent.isLayoutRequested()) {
        //由此向ViewParent请求布局
        //从这个View开始向上一直requestLayout，最终到达ViewRootImpl的requestLayout
        mParent.requestLayout();
    }
    ......
}
```
和invalidate相同，也是逐级向上，直到ViewRootImpl的requestLayout方法
```Java
public void requestLayout() {
   if (!mHandlingLayoutInLayoutRequest) {
       checkThread();
       mLayoutRequested = true;
       //View调运requestLayout最终层层上传到ViewRootImpl后最终触发了该方法
       scheduleTraversals();
   }
}
```
这也就明白了。不过要注意，requestLayout()方法会调用measure过程和layout过程，不会调用draw过程，也不会重新绘制任何View包括该调用者本身。


参考：http://blog.csdn.net/yanbober/article/details/46128379/
