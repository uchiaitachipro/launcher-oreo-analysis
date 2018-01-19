# DragLayer分析
----

## 1. 概述
DragLayer主要提供了两个功能：
1. [捕获Touch事件，结合DragController实现Launcher内拖拽效果。](./DragAnalysis.md)
2. [DragView放下动画。](./DragLayerAnim.md)

当然DragLayer的Touch事件也不仅仅是被拖拽消费掉。在onInterceptTouchEvent函数里包括了各种消耗Touch事件的情况。
1. AppWidgetResizeFrame.onControllerInterceptTouchEvent 是判断是否为桌面长按小部件调整大小。
2. mDragController.onControllerInterceptTouchEvent 是判断是否为拖拽图标等。
3. mAllAppsController.onControllerInterceptTouchEvent 是判断是否为拉起应用页。
4. widgetsBottomSheet.onControllerInterceptTouchEvent 是判断是否应用创建小部件的WidgetsBottomSheet需要创建小部件或者消失。参见[固定快捷方式和小部件](https://developer.android.google.cn/about/versions/oreo/android-8.0.html)
5. PinchToOverviewListener.onControllerInterceptTouchEvent 是判断是否为创建Pinch Shortcut。 参见[App Shortcuts](https://developer.android.google.cn/about/versions/oreo/android-8.0.html)

DragLayer也提供了子View在DragLayer中的位置，这是实现拖拽基础。

## 2. DragLayer、DragView、DragController之间关系
用一句话来概括就是DragLayer捕获Touch事件，DragController将事件转发到对应的DropTarget中，DragView显示被拖拽View的位置。
需要注意的一点是在Launcher3-oreo中DropTarget处理拖拽具体逻辑已经交给DragDriver而不是DragController自己处理了。在DragController.startDrag会创建DragDriver并在DragController.onControllerTouchEvent将Touch事件委托给DragDriver处理。

## 3. DragView 分析
### 3.1 DragView拖动分析
在DragView中有几个和拖动有关的重要变量mRegistrationX mRegistrationY mLastTouchX mLastTouchY mAnimatedShiftX mAnimatedShiftY。  

|变量     |作用     |
| ------- | :-----: |
| mRegistrationX<br>mRegistrationX | 是记录长按拖拽图标的初始位置。|
| mLastTouchX<br>mLastTouchY | 是当前Touch事件中的 x,y 坐标。|
| mAnimatedShiftX<br>mAnimatedShiftY | 是PinchShortcut的长按的的初始位置距离DragView的中心偏移。|  

首先来看DragView的move函数
```java {.line-numbers}

    public void move(int touchX, int touchY) {
        mLastTouchX = touchX;
        mLastTouchY = touchY;
        applyTranslation();
    }

    private void applyTranslation() {
        setTranslationX(mLastTouchX - mRegistrationX + mAnimatedShiftX);
        setTranslationY(mLastTouchY - mRegistrationY + mAnimatedShiftY);
    }

```
_DragView.java_

mRegistrationX mRegistrationX mLastTouchX mLastTouchY具体作用如下图所示：  
![DragView](./images/DragView.png)  
其中mAnimatedShiftX mAnimatedShiftY一般为0，只有拖拽应用快捷方式时才会有值，并且记录的是当前触摸坐标与DragView中心的偏移。
```java {.line-numbers}

    @Override
    public boolean onTouch(View v, MotionEvent ev) {
        // Touched a shortcut, update where it was touched so we can drag from there on long click.
        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
            case MotionEvent.ACTION_MOVE:
                mIconLastTouchPos.set((int) ev.getX(), (int) ev.getY());
                break;
        }
        return false;
    }

    @Override
    public boolean onLongClick(View v) {

        // ...
          // Move the icon to align with the center-top of the touch point
        mIconShift.x = mIconLastTouchPos.x - sv.getIconCenter().x;
        mIconShift.y = mIconLastTouchPos.y - mLauncher.getDeviceProfile().iconSizePx;
        // ...
        dv.animateShift(-mIconShift.x, -mIconShift.y);
        // ..
    }

```
_ShortcutItemView.java_

```java {.line-numbers}

    public void animateShift(final int shiftX, final int shiftY) {
        if (mAnim.isStarted()) {
            return;
        }
        mAnimatedShiftX = shiftX;
        mAnimatedShiftY = shiftY;
        applyTranslation();
        mAnim.addUpdateListener(new AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                float fraction = 1 - animation.getAnimatedFraction();
                mAnimatedShiftX = (int) (fraction * shiftX);
                mAnimatedShiftY = (int) (fraction * shiftY);
                applyTranslation();
            }
        });
    }

```
_DragView.java_
由DragView.animateShift可知当ShortItemView长按生成DragView的中心最后会和指间重合同时mAnimatedShiftX mAnimatedShiftY变为0。由此可知一般来说DragView.applyTranslation 中mAnimatedShiftX mAnimatedShiftY是为了偏移到中心而设的。  
**如果Shortcut、Folder也需要在长按时将其中心偏移到指间中心，也可以采用ShortcutsItemView一样的方法** 

### 3.2 Oreo拖拽图标内容晃动的动画效果分析
//TODO
## 4. DragLayer.onInterceptTouchEvent切换mActiveController分析
### 4.1 小部件调整大小分析
Step1. 当长按小部件然后放下时，AppWidgetResizeFrame会被添加到对应的小部件的位置。  
Step2. 再次按下若触摸区域在AppWidgetResizeFrame内，则mActiveController为
AppWidgetResizeFrame。

拖拽小部件并放下,并在onDropCompleted执行
```java {.line-numbers}

    public void onDrop(final DragObject d) {
        //...
        if (pInfo != null && pInfo.resizeMode != AppWidgetProviderInfo.RESIZE_NONE
            && !d.accessibleDrag) {
            mDelayedResizeRunnable = new Runnable() {
                public void run() {
                    if (!isPageInTransition()) {
                        DragLayer dragLayer = mLauncher.getDragLayer();
                        dragLayer.addResizeFrame(hostView, cellLayout);
                    }
                }
            };
        }
        //..
    }

    public void onDropCompleted(final View target, final DragObject d,
            final boolean isFlingToDelete, final boolean success) {

        //...
        if (!isFlingToDelete) {
            // Fling to delete already exits spring loaded mode after the animation finishes.
            mLauncher.exitSpringLoadedDragModeDelayed(success,
                    Launcher.EXIT_SPRINGLOADED_MODE_SHORT_TIMEOUT, mDelayedResizeRunnable);
            mDelayedResizeRunnable = null;
        }
    }

```  
_Workspace.java_  
  
AppWidgetResizeFrame调整布局和小部件所在区域重合  
```java {.line-numbers}

    public void addResizeFrame(LauncherAppWidgetHostView widget, CellLayout cellLayout) {
        clearResizeFrame();

        mCurrentResizeFrame = (AppWidgetResizeFrame) LayoutInflater.from(mLauncher)
                .inflate(R.layout.app_widget_resize_frame, this, false);
        mCurrentResizeFrame.setupForWidget(widget, cellLayout, this);
        ((LayoutParams) mCurrentResizeFrame.getLayoutParams()).customPosition = true;

        addView(mCurrentResizeFrame);
        mCurrentResizeFrame.snapToWidget(false);
    }

```  
_DragLayer.java_  
```java {line-numbers}


    /**
     * Returns the rect of this view when the frame is snapped around the widget, with the bounds
     * relative to the {@link DragLayer}.
     */
    private void getSnappedRectRelativeToDragLayer(Rect out) {
        float scale = mWidgetView.getScaleToFit();

        mDragLayer.getViewRectRelativeToSelf(mWidgetView, out);

        int width = 2 * mBackgroundPadding
                + (int) (scale * (out.width() - mWidgetPadding.left - mWidgetPadding.right));
        int height = 2 * mBackgroundPadding
                + (int) (scale * (out.height() - mWidgetPadding.top - mWidgetPadding.bottom));

        int x = (int) (out.left - mBackgroundPadding + scale * mWidgetPadding.left);
        int y = (int) (out.top - mBackgroundPadding + scale * mWidgetPadding.top);

        out.left = x;
        out.top = y;
        out.right = out.left + width;
        out.bottom = out.top + height;
    }

    public void snapToWidget(boolean animate) {
        getSnappedRectRelativeToDragLayer(sTmpRect);
        //... 
        final DragLayer.LayoutParams lp = (DragLayer.LayoutParams) getLayoutParams();
        if (!animate) {
            lp.width = newWidth;
            lp.height = newHeight;
            lp.x = newX;
            lp.y = newY;
            for (int i = 0; i < HANDLE_COUNT; i++) {
                mDragHandles[i].setAlpha(1.0f);
            }
            requestLayout();
        }
        //...
    }

```  
_AppWidgetResizeFrame.java_  

**判断TouchEvent是否落在AppWidgetResizeFrame内，若是则为mActiveController为AppWidgetResizeFrame**
```java {.line-numbers}

    @Override
    public boolean onControllerInterceptTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN && handleTouchDown(ev)) {
            return true;
        }
        return false;
    }

    private boolean handleTouchDown(MotionEvent ev) {
        Rect hitRect = new Rect();
        int x = (int) ev.getX();
        int y = (int) ev.getY();

        getHitRect(hitRect);
        if (hitRect.contains(x, y)) {
            if (beginResizeIfPointInRegion(x - getLeft(), y - getTop())) {
                mXDown = x;
                mYDown = y;
                return true;
            }
        }
        return false;
    }

```  
_AppWidgetResizeFrame.java_  

### 4.2 DragController为mActiveController条件
当快捷方式、文件夹、小部件长按时会在DragController.startDrag创建mDragDriver时，就会拦截。
```java {.line-numders}

    public DragView startDrag(Bitmap b, int dragLayerX, int dragLayerY,
                DragSource source, ItemInfo dragInfo, Point dragOffset, Rect dragRegion,
                float initialDragViewScale, DragOptions options) {
                    //...
                    mDragDriver = DragDriver.create(mLauncher, this, mDragObject, mOptions);
                    //... 

    }

    public boolean onControllerInterceptTouchEvent(MotionEvent ev) {
        //...
        return mDragDriver != null && mDragDriver.onInterceptTouchEvent(ev);
    }
```  
_DragController.java_  

```java {.line-numbers}

    public boolean onInterceptTouchEvent(MotionEvent ev) {
        //...
        return true;
    }

```
_DragDriver.java_  

### 4.3 应用菜单拦截条件
```java {.line-numbers}

    @Override
    public boolean onControllerInterceptTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            mNoIntercept = false;
            mTouchEventStartedOnHotseat = mLauncher.getDragLayer().isEventOverHotseat(ev);
            //Condition A
            if (!mLauncher.isAllAppsVisible() && mLauncher.getWorkspace().workspaceInModalState()) {
                mNoIntercept = true;
            }
            //Condition B 
            else if (mLauncher.isAllAppsVisible() &&
                    !mAppsView.shouldContainerScroll(ev)) {
                mNoIntercept = true;
            } 
            //Condition C
            else if (AbstractFloatingView.getTopOpenView(mLauncher) != null) {
                mNoIntercept = true;
            } else {
                //...
                //Condition D 
        }

        if (mNoIntercept) {
            return false;
        }
        mDetector.onTouchEvent(ev);

        //Condition E
        if (mDetector.isSettlingState() && (isInDisallowRecatchBottomZone() || isInDisallowRecatchTopZone())) {
            return false;
        }
        return mDetector.isDraggingOrSettling();
    }

```
_AllAppsTransitionController.java_  
**Condition A.** 当进入预览模式，点击hotseat区域不处理。  
**Condition B.** 当应用列表处于打开状态并且里面的内容不能滚动(例如应用列表里的应用只有几个不满一屏的情况)。  
**Condition C.** 例如PopupContainerWindowWithArrow处于打开状态，Touch事件就不由mAllAppsController处理。  
**Condition D.** 处理应用列表上拉下拉的情况和处理应用列表处于上拉下滑动画过程中，再次上拉下滑操作。通常是取消原来的操作。  
**Condition E.** 处理底部顶部临界区情况，参见``java RECATCH_REJECTION_FRACTION ``定义。  

### 4.4 其他条件
//TODO