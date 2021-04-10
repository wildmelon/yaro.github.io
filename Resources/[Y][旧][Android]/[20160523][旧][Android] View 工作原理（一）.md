## 备注
**原发表于2016.05.23，资料已过时，仅作备份，谨慎参考**

## 前言
本文参考《Android 开发艺术探索》及网上各种资料进行撰写，目的是为自己理清 Android 中 View 的工作原理，复习学习内容，为后期阅读开源自定义 View 源码做好准备，深入学习可查看参考资料中的内容。

## 基本概念
本节介绍两个基本概念，为理解后面小节内容预热。
### DecorView
DecorView 是 Window 中 View 的顶层 View，其结构如下所示：

![](http://upload-images.jianshu.io/upload_images/1903766-2ea4ac02adfd2af3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

DecorView 其实是一个 FrameLayout，其中包含了一个 LinearLayout，分为上下部分（两个 FrameLayout）。

我们在 setContentView 所设置的布局文件就是被加到下部分中 android.R.id.content 的 FrameLayout 中。

### ViewRoot
ViewRoot 对应于 ViewRootImpl 类，是连接 WindowManager 和 DecorView 的纽带。

当 Activity 对象创建完毕后，会将 DecorView 添加到 Window 中，同时创建 ViewRootImpl 对象与 DecorView 建立关联。

View 的绘制流程是从 ViewRoot 的 performTraversals 方法开始的，其工作流程图如下所示：

![](http://upload-images.jianshu.io/upload_images/1903766-d96e60cfa596e250.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

performTraversals 方法的代码十分的长，我们先只看其中一小部分：

    private void performTraversals() {
      ...
      if (!mStopped) {
        boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
        if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                || mHeight != host.getMeasuredHeight() || contentInsetsChanged) {
            
            // 获取测量规格，mWidth 和 mHeight 当前视图 frame 的大小
            // lp是WindowManager.LayoutParams
            int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
            int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
    
             // Ask host how big it wants to be
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            ...
          }
          ...
        }
    }

这里调用了 performMeasure 开始进行测量，传入了两个 MeasureSpec 参数，我们先来学习一下 MeasureSpec 的概念。

## MeasureSpec
### 概念
MeasureSpec 参与了 View 的 measure 过程，系统根据 MeasureSpec 来测量出 View 的测量宽/高。

- 对于 DecorView，其 MeasureSpec 是由窗口尺寸和自身的 LayoutParams 来共同确定的
- 对于普通 View，是由父容器的 MeasureSpec 和自身的 LayoutParams 来共同确定的

一个 MeasureSpec 由 SpecMode 和 SpecSize 组成，分别指测量模式和规格大小。

SpecMode 有三种模式：

1. UNSPECIFIED，父容器不对 View 有所限制，要多大给多大，一般用于系统内部。
2. EXACTLY，父容器检测出 View 所需的大小，这时 View 的最终大小就是 SpecSize 所指定的值。它对应于 View LayoutParmas 中的 match_parent 和具体数值。
3. AT_MOST，父容器指定了一个可用的大小，View 的大小不能大于该 SpecSize。它对应于 LayoutParams 中的 wrap\_content。

### DecorView 的 MeasureSpec
对于 DecorView，在 ViewRootImpl.measureHierarchy() 方法中，有如下代码：

    childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
    childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

其中传入的参数 desiredWindowWidth 和 desiredWindowHeight 是屏幕的尺寸，下面再看一下 getRootMeasureSpec 的实现：

    private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }

即是，根据 DecorView 的 LayoutParams 的宽/高的值，有以下情况

1. LayoutParams.MATCH_PARENT：DecorView 确定宽/高为窗口大小
2. LayoutParams.WRAP_CONTENT:大小不定，不能超出窗口大小
3. 固定大小：确定宽/高为 LayoutParams 指定的大小

接着返回 measureSpec 以供下一步测量使用。performMeasure 方法会从 DecorView 开始，逐层往下进行测量。

### 普通 View 的 MeasureSpec
对普通的 View 的 measure 方法的调用，是由其父容器传递而来的，这里先看一下 ViewGroup 的 measureChildWithMargins 方法：

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

这里先获取了 子View 的 LayoutParams，与父容器的 MeasureSpec 一起生成了 子View 的 MeasureSpec，再调用 View 的 measure 方法。

再继续看，子View 的 MeasureSpec 是如何生成的：

    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }

尽管代码稍微长一点，但逻辑还是很简单的，先将 size 减去 padding 和 margin 占用的控件，再根据父容器的 SpecMode 和 View 设置的 LayoutParams 来确定 View 的 measureSpec。

上述代码可简化为下图所示：

![](http://upload-images.jianshu.io/upload_images/1903766-e2a47200df7d90d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

例如如果父容器的 measureSpec 是 AT_MOST 模式，View 的 LayoutParams 是 MATCH_PARENT，则 View 的 measureSize 为父容器可用大小，measureMode 与父容器相同为 AT_MOST。

上述表格是可凭逻辑进行推断的，所以只要看懂代码，无需死记硬背。另外 UNSPECIFIED 模式一般用于系统内部，故不需过多关注。

## 参考资料
[View测量机制详解—从DecorView说起](http://blog.csdn.net/qibin0506/article/details/49245601)

[android:padding和android:margin的区别](http://blog.csdn.net/xxdbupt/article/details/20450915)