# Android视图绘制原理
Android系统的视图结构的设计也采用了组合模式，即View作为所有图形的基类，Viewgroup对View继承扩展为视图容器类，由此就得到了视图部分的基本结构--树形结构<br>
![image](https://github.com/zhuyuezong/View-Draw/blob/master/images/1343827041_4739.png?raw=true)<br>



```Android
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
