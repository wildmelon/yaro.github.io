## 备注
**原发表于2016.05.27，资料已过时，仅作备份，谨慎参考**

## 前言
本文大量参照《Android 开发艺术探索》及参考资料的内容整合，主要帮助自己理清 View 的工作原理。深入学习希望大家更多的关注参考资料。

上一篇文章了解了 MeasureSpec 的概念及获取，从名字上看就能了解到这是用来辅助测量过程的对象，本次文章再来完整学习 View 的工作流程。

View 的工作流程主要指 measure、layout、draw 这三个过程：

- measure：确定 View 的测量宽/高
- layout：确定 View 的最终宽/高和四个顶点位置
- draw：将 View 绘制到屏幕上

![](http://upload-images.jianshu.io/upload_images/1903766-506e1399dcea498f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## measure
如果是一个原始的 View，那么通过 measure 方法就完成了其测量过程。

如果是一个 ViewGroup，那么除了完成自己的测量过程之外，还会遍历去调用所有子元素的 measure()。

### View 的 measure 过程
View 的 measure 过程由其 measure 方法完成，在 measure() 中会调用 onMeasure() 方法，这是实际测量的地方，代码如下：

    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }

setMeasuredDimension 会设置 View 的测量宽高，所以只需看 getDefaultSize 方法即可：

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

可以看到逻辑十分简单，当 measureSpec 的模式为 AT_MOST 或 EXACTLY 时，specSize 即为 View 测量后的大小。

而当为 UNSPECIFIED，View 测量大小为 getSuggestedMinimumWidth() 的值，会根据 View 的 android:minWidth 和 background 属性获取一个值，由于这种情况一般用于系统内部，所以就不深究，源码十分简单，大家可以亲自动手一试。

另外这里可以留意到，当你的 View 的宽高设置为 wrap\_content 时。该 View 的 specSize 为 parentSize，specMode 为 AT_MOST；结果根据上面代码 View 的测量宽高就是父容器的宽高，这样的结果跟 match\_parent 就没有区别！

所以实际上像 TextView 这些控件，都会重写 onMeasure 方法，根据 View 的 LayoutParams 为 wrap\_content 的情况设置一个宽高，通常是小于父容器的。

### ViewGroup 的 measure 过程
ViewGroup 是一个继承自 View 的抽象类，它并没有重写 View 的 onMeasure 方法，而是由其抽象实现类例如 LinearLayout 来重写，因为不同的 ViewGroup 有不同的特性。

ViewGroup 提供了一个 measureChildren 方法，代码如下：

    protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }

就是调用 measureChild 传入每一个子元素：

    protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }

这里 getChildMeasureSpec 方法就十分熟悉了，根据子元素的 LayoutParams 和 父容器的 measureSpec 创建子元素的 measureSpec，然后调用子元素的 measure 方法，这样就是上一节的内容了。

如此递归完成了 View 的测量过程。

## layout
layout 的作用是确定元素的位置，当 ViewGroup 调用 layout 方法确定自己的位置后，又会调用 onLayout 来确定所有子元素的位置。

### View 的 layout 过程
实际上 ViewGroup.layout() 中会通过 super.layout 调用其父类，即 View 的 layout 方法，代码如下：

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

一些判断和代码细节可以略过，这里主要有两句，一是调用 setFrame 方法确定了 View 的四个顶点位置，接着调用 onLayout 方法，该方法负责父容器确定子元素的位置。所以 View 和 ViewGroup 的 onLayout 都没有进行具体的实现，因为不同的布局有不同的特性，我们下面来看一下 LinearLayout 的实现。

### LinearLayout 的 onLayout 过程
直接上代码~

    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        if (mOrientation == VERTICAL) {
            layoutVertical(l, t, r, b);
        } else {
            layoutHorizontal(l, t, r, b);
        }
    }

这个 mOrientation 很明显了，我们只看垂直方向的 layoutVertical 方法，代码比较长：

    void layoutVertical(int left, int top, int right, int bottom) {
        ...

        for (int i = 0; i < count; i++) {
            final View child = getVirtualChildAt(i);
            if (child == null) {
                childTop += measureNullChild(i);
            } else if (child.getVisibility() != GONE) {
                final int childWidth = child.getMeasuredWidth();
                final int childHeight = child.getMeasuredHeight();
                
                final LinearLayout.LayoutParams lp =
                        (LinearLayout.LayoutParams) child.getLayoutParams();
                ...

                if (hasDividerBeforeChildAt(i)) {
                    childTop += mDividerHeight;
                }

                childTop += lp.topMargin;
                setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                        childWidth, childHeight);
                childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

                i += getChildrenSkipCount(child, i);
            }
        }
    }

去除了一些细节内容，LinearLayout 的 layoutVertical 会调用 setChildFrame 来为子元素指定对应的位置；同时变量 childTop 会不断增加，使得后面的元素位置逐渐靠下，跟我们平时使用 LinearLayout 的效果特性是符合的。

setChildFrame 方法的代码如下：

    private void setChildFrame(View child, int left, int top, int width, int height) {        
        child.layout(left, top, left + width, top + height);
    }

调用了子元素的 layout 方法，这样又是逐步递归调用，完成 View 的布局流程。

## draw
Draw 的作用是把 View 绘制到屏幕上来。绘制时的流程图如下：

![](http://upload-images.jianshu.io/upload_images/1903766-77957a658bb20d5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体再来看看 draw 方法的代码：

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
        ...
    }

其中逐步执行了流程图中的内容，并且在不需要时会跳过 layer 的绘制以提高效率。

在 View 中，onDraw() 和 dispatchDraw() 都为空实现，因为不同的控件会有不同的绘制方式。于是我们也意识到，自定义绘制过程需要复写 onDraw 方法来绘制自身的内容。

而 ViewGroup 对 dispatchDraw() 则进行了实现，在其中通过 drawChild() 调用了子元素的 draw 方法，又是递归完成了绘制过程。

这里涉及到动画等内容的绘制，比较复杂，与理解 View 的工作原理的关系不强，所以可能在以后学习一些简单的自定义控件时再来学习控件是通过何种方式进行绘制的。

以上便是 View 的工作流程，再次声明，大量参考学习了他人的文章内容。

希望能对大家有所帮助，感谢这些优秀文章的作者们。

## 参考资料
[公共技术点之 View 绘制流程](http://a.codekk.com/detail/Android/lightSky/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20View%20%E7%BB%98%E5%88%B6%E6%B5%81%E7%A8%8B)

[Android应用层View绘制流程与源码分析](http://blog.csdn.net/yanbober/article/details/46128379/)