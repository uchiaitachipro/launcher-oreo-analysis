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
mRegistrationX mRegistrationY 是记录长按拖拽图标的初始位置。  
mLastTouchX mLastTouchY 是当前Touch事件中的 x,y 坐标。 
mAnimatedShiftX mAnimatedShiftY 是PinchShortcut的长按的的初始位置距离DragView的中心偏移。


