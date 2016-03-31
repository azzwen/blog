---
title: LinearLayout onMeasure 过程分析
date: 2016-03-17 15:43:12
tags:
	- Android
	- LinearLayout
	- measure
---

`LinearLayout` 是最常用的布局之一，但某些属性很令人费解。为了更加清楚的认识 `LinearLayout` ，也为了加深对 `ViewGroup` 测量过程的理解，决定从源码角度研究一下测量过程。

<!-- more -->

> 分析使用版本 6.0.1 r22 
> 源码地址 [`LinearLayout`](https://android.googlesource.com/platform/frameworks/base/+/android-6.0.1_r22/core/java/android/widget/LinearLayout.java)


## onMeasure
``` java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    if (mOrientation == VERTICAL) {
       measureVertical(widthMeasureSpec, heightMeasureSpec);
    } else {
        measureHorizontal(widthMeasureSpec, heightMeasureSpec);
    }
}
```
`onMeasure()` 只是根据 `LinearLayout` 的方向调用不同的测量方法。下面以 `measureVertical()` 为例进行分析。

## measureVertical
``` java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    mTotalLength = 0;
    int maxWidth = 0;
    int childState = 0;
    int alternativeMaxWidth = 0;
    int weightedMaxWidth = 0;
    boolean allFillParent = true;
    float totalWeight = 0;

    final int count = getVirtualChildCount();

    final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);

    boolean matchWidth = false;
    boolean skippedMeasure = false;

    final int baselineChildIndex = mBaselineAlignedChildIndex;        
    final boolean useLargestChild = mUseLargestChild;

    int largestChildHeight = Integer.MIN_VALUE;
    
    // *****************************************
    // * 第一次测量过程
    // *****************************************
    // See how tall everyone is. Also remember max width.
    for (int i = 0; i < count; ++i) {
        ......
    }

    // 判断是否在 LinearLayout 底部添加分隔符
    if (mTotalLength > 0 && hasDividerBeforeChildAt(count)) {
        mTotalLength += mDividerHeight;
    }

    if (useLargestChild &&
            (heightMode == MeasureSpec.AT_MOST || heightMode == MeasureSpec.UNSPECIFIED)) {
        // 如果设置了 measureWithLargestChild 且 高度测量模式为 AT_MOST 或者 UNSPECIFIED，则重新计算 mToatalLength
        // 具体计算方式见下文分析二
        mTotalLength = 0;
        for (int i = 0; i < count; ++i) {
            ......
        }
    }

    mTotalLength += mPaddingTop + mPaddingBottom;
    int heightSize = mTotalLength;
    heightSize = Math.max(heightSize, getSuggestedMinimumHeight());

    // 根据高度测量规格计算最终高度
    // heightSizeAndState 就是 LinearLayout 最终测量结果
    int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
    heightSize = heightSizeAndState & MEASURED_SIZE_MASK;

    // Either expand children with weight to take up available space or
    // shrink them if they extend beyond our current bounds. If we skipped
    // measurement on any children, we need to measure them now.
    int delta = heightSize - mTotalLength;
    // 根据之前的测量结果，重新测量子视图大小
    if (skippedMeasure || delta != 0 && totalWeight > 0.0f) {
        ......
    } else {
        ......
    }

    if (!allFillParent && widthMode != MeasureSpec.EXACTLY) {
        maxWidth = alternativeMaxWidth;
    }

    maxWidth += mPaddingLeft + mPaddingRight;

    // Check against our minimum width
    maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            heightSizeAndState);

    if (matchWidth) {
        forceUniformWidth(count, heightMeasureSpec);
    }
}
```

## 第一次测量过程
测量所有子视图的高度，并且记录宽度相关的测量数据。关于宽度测量，详见分析六。
``` java
// See how tall everyone is. Also remember max width.
for (int i = 0; i < count; ++i) {
    final View child = getVirtualChildAt(i);

    // 测量为 null 的子视图的高度，目前返回 0，用于以后扩展
    if (child == null) {
        mTotalLength += measureNullChild(i);
        continue;
    }
    // getChildrenSkipCount() 计算跳过的子视图个数，返回 0
    if (child.getVisibility() == View.GONE) {
       i += getChildrenSkipCount(child, i);
       continue;
    }
    // 将分割线的高度计算在 mTotalLength 内
    if (hasDividerBeforeChildAt(i)) {
        mTotalLength += mDividerHeight;
    }

    LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams) child.getLayoutParams();

    totalWeight += lp.weight;

    if (heightMode == MeasureSpec.EXACTLY && lp.height == 0 && lp.weight > 0) {
        // 满足该条件的话，不需要现在计算该子视图的高度。测量工作会在之后进行
        // 为什么满足这三个条件的不用计算高度？见下文分析一
        // Optimization: don't bother measuring children who are going to use
        // leftover space. These views will get measured again down below if
        // there is any leftover space.
        final int totalLength = mTotalLength;
        mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
        skippedMeasure = true;
    } else {
        int oldHeight = Integer.MIN_VALUE;

        if (lp.height == 0 && lp.weight > 0) {
            // 如果 LiniearLayout 的高度需要根据子视图来测量，为了测量子视图的高度，设置子视图 LayoutParams.height 为 wrap_content。
            // heightMode is either UNSPECIFIED or AT_MOST, and this
            // child wanted to stretch to fill available space.
            // Translate that to WRAP_CONTENT so that it does not end up
            // with a height of 0
            oldHeight = 0;
            lp.height = LayoutParams.WRAP_CONTENT;
        }

        // Determine how big this child would like to be. If this or
        // previous children have given a weight, then we allow it to
        // use all available space (and we will shrink things later
        // if needed).
        // 只是调用了 ViewGroup 的 measureChildWithMargins() 对子视图进行测量
        // 第四个参数表示当前已使用的宽度，因为是纵向 LinearLayout ，所以一直为 0 。
        // 第六个参数表示已使用的高度。如果之前子视图或者当前的子视图有 weight 属性，就允许当前子视图使用 LinearLayout 的所有高度，即已使用的高度为 0 。
        measureChildBeforeLayout(
               child, i, widthMeasureSpec, 0, heightMeasureSpec,
               totalWeight == 0 ? mTotalLength : 0);

        if (oldHeight != Integer.MIN_VALUE) {
           // 测量完成之后，重新设置 LayoutParams.height
           lp.height = oldHeight;
        }

        final int childHeight = child.getMeasuredHeight();
        final int totalLength = mTotalLength;
        // 重新计算 mTotalLength
        mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
               lp.bottomMargin + getNextLocationOffset(child));
        // 设置最高子视图大小
        if (useLargestChild) {
            largestChildHeight = Math.max(childHeight, largestChildHeight);
        }
    }

    /**
     * If applicable, compute the additional offset to the child's baseline
     * we'll need later when asked {@link #getBaseline}.
     */
     // mBaselineChildTop 表示指定的 baseline 的子视图的顶部高度
    if ((baselineChildIndex >= 0) && (baselineChildIndex == i + 1)) {
       mBaselineChildTop = mTotalLength;
    }

    // 设置为 baseline 的子视图的前面不允许设置 weiget 属性
    // if we are trying to use a child index for our baseline, the above
    // book keeping only works if there are no children above it with
    // weight.  fail fast to aid the developer.
    if (i < baselineChildIndex && lp.weight > 0) {
        throw new RuntimeException("A child of LinearLayout with index "
                + "less than mBaselineAlignedChildIndex has weight > 0, which "
                + "won't work.  Either remove the weight, or don't set "
                + "mBaselineAlignedChildIndex.");
    }

    // 以下代码进行宽度相关的测量工作
    boolean matchWidthLocally = false;
    if (widthMode != MeasureSpec.EXACTLY && lp.width == LayoutParams.MATCH_PARENT) {
        // The width of the linear layout will scale, and at least one
        // child said it wanted to match our width. Set a flag
        // indicating that we need to remeasure at least that view when
        // we know our width.
        matchWidth = true;
        matchWidthLocally = true;
    }

    final int margin = lp.leftMargin + lp.rightMargin;
    final int measuredWidth = child.getMeasuredWidth() + margin;
    maxWidth = Math.max(maxWidth, measuredWidth);
    childState = combineMeasuredStates(childState, child.getMeasuredState());

    allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;
    if (lp.weight > 0) {
        /*
         * Widths of weighted Views are bogus if we end up
         * remeasuring, so keep them separate.
         */
        weightedMaxWidth = Math.max(weightedMaxWidth,
                matchWidthLocally ? margin : measuredWidth);
    } else {
        alternativeMaxWidth = Math.max(alternativeMaxWidth,
                matchWidthLocally ? margin : measuredWidth);
    }

    i += getChildrenSkipCount(child, i);
}
```
**分析一：为什么不用计算某些子视图高度？**

- 子视图的 `height` 为 0，`weight` 不为 0，说明该视图**仅仅**希望使用 `LinearLayout` 的剩余空间。
- `LinearLayout` 的高度测量规格为 `Exactly` ，说明 `LinearLayout` 的高度已经确定，不依赖子视图的高度计算自己的高度。相反，如果测量规格为 `AT_MOST` 或者 `UNSPECIFIED` ，`LinearLayout` 只能根据子视图的高度来确定自己的高度，就必须对所有的子视图进行测量。

## 重新计算 mTotalLength
``` java
for (int i = 0; i < count; ++i) {
    final View child = getVirtualChildAt(i);

    if (child == null) {
        mTotalLength += measureNullChild(i);
        continue;
    }

    if (child.getVisibility() == GONE) {
        i += getChildrenSkipCount(child, i);
        continue;
    }

    final LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams)
            child.getLayoutParams();
    // Account for negative margins
    final int totalLength = mTotalLength;
    // 计算所有子视图高度之和
    mTotalLength = Math.max(totalLength, totalLength + largestChildHeight +
            lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
}
```
**分析二：如何计算  `mTotalLength` ？**

`mTotalLength` 为已测量的子视图的总高度。如果设置了 `android:measureWithLargestChild="true"` ，每个子视图的高度为：最大子视图高度 ＋ 该子视图的上下外边距。

> 注意，重新计算 `mTotalLength` 时，没有将 **间隔线的高度** 计算在内。造成设置 `android:measureWithLargestChild="true"` 且显示间隔线时，`LinearLayout` 高度计算会出现问题。
> 为什么会出现这个明显的问题呢？难道设计时，这两个属性就不会同时出现？
 
**分析三：`measureWithLargestChild` 如何影响 `LinearLayout` 高度的测量？**
首先来看对 `LinearLayout` 高度测量的影响。
根据源码可知：
- `LinearLayout` 的测量高度保存在 `heightSizeAndState`。
- `heightSizeAndState` 的值由 `mTotalLength`  和 `getSuggestedMinimumHeight()` `resolveSizeAndState`  方法决定。
- 重新测量 `mTotalLength` 的触发条件为：`useLargestChild && heightMode != Exactly`

考虑以下布局
``` xml
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:measureWithLargestChild="true"
    android:background="@android:color/darker_gray"
    android:orientation="vertical">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:background="@android:color/holo_red_dark"
        android:textSize="15sp"
        android:text="TextView1"/>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:background="@android:color/holo_red_light"
        android:textSize="20sp"
        android:text="TextView2"/>
</LinearLayout>
```
 预览效果如下：
![](/assets/images/LinearLayout onMeasure 过程分析/linearlayout_test02.png)
可以看出， `LinearLayout` 的高度比子视图总高度多出一部分。
测量过程分析：
1. 第一次测量时，两个子视图都会进行测量，并记录 `mTotalLength`。
2. 重新计算 `mTotalLength` = 子视图个数 * 子视图最大高度 。
3. 重新测量子视图大小时，执行 `else` 分支。但是，子视图没有 `weight` 属性，不会再次测量，子视图维持之前的测量大小。
4. `LinearLayout` 的高度使用了 **子视图最大高度** ，但是子视图没有进行拉伸，造成空间剩余。

## 重新测量子视图大小
根据 weight 分配 `LinearLayout` 的剩余高度。`LinearLayout` 的剩余高度可能为负值。见下文分析三。
``` java
if (skippedMeasure || delta != 0 && totalWeight > 0.0f) {
    // 注意，delta 可能小于零
    float weightSum = mWeightSum > 0.0f ? mWeightSum : totalWeight;

    mTotalLength = 0;

    for (int i = 0; i < count; ++i) {
        final View child = getVirtualChildAt(i);
        
        if (child.getVisibility() == View.GONE) {
            continue;
        }
        
        LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams) child.getLayoutParams();
        
        float childExtra = lp.weight;
        if (childExtra > 0) {
            // Child said it could absorb extra space -- give him his share
            // 计算 weight 属性分配的大小，可能为负值
            int share = (int) (childExtra * delta / weightSum);
            weightSum -= childExtra;
            delta -= share;

            final int childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                    mPaddingLeft + mPaddingRight +
                            lp.leftMargin + lp.rightMargin, lp.width);

            if ((lp.height != 0) || (heightMode != MeasureSpec.EXACTLY)) {
                // 子视图已经测量过
                // child was measured once already above...
                // base new measurement on stored values
                int childHeight = child.getMeasuredHeight() + share;
                if (childHeight < 0) {
                    childHeight = 0;
                }
                
                child.measure(childWidthMeasureSpec,
                        MeasureSpec.makeMeasureSpec(childHeight, MeasureSpec.EXACTLY));
            } else {
                // 子视图第一次测量
                // child was skipped in the loop above.
                // Measure for this first time here      
                child.measure(childWidthMeasureSpec,
                        MeasureSpec.makeMeasureSpec(share > 0 ? share : 0,
                                MeasureSpec.EXACTLY));
            }

            // Child may now not fit in vertical dimension.
            childState = combineMeasuredStates(childState, child.getMeasuredState()
                    & (MEASURED_STATE_MASK>>MEASURED_HEIGHT_STATE_SHIFT));
        }
        // 处理子视图宽度
        final int margin =  lp.leftMargin + lp.rightMargin;
        final int measuredWidth = child.getMeasuredWidth() + margin;
        maxWidth = Math.max(maxWidth, measuredWidth);

        boolean matchWidthLocally = widthMode != MeasureSpec.EXACTLY &&
                lp.width == LayoutParams.MATCH_PARENT;

        alternativeMaxWidth = Math.max(alternativeMaxWidth,
                matchWidthLocally ? margin : measuredWidth);

        allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;

        final int totalLength = mTotalLength;
        mTotalLength = Math.max(totalLength, totalLength + child.getMeasuredHeight() +
                lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
    }

    // Add in our padding
    mTotalLength += mPaddingTop + mPaddingBottom;
} else {
    alternativeMaxWidth = Math.max(alternativeMaxWidth,
                                   weightedMaxWidth);


    // We have no limit, so make all weighted views as tall as the largest child.
    // Children will have already been measured once.
    if (useLargestChild && heightMode != MeasureSpec.EXACTLY) {
        for (int i = 0; i < count; i++) {
            final View child = getVirtualChildAt(i);
            if (child == null || child.getVisibility() == View.GONE) {
                continue;
            }
            final LinearLayout.LayoutParams lp =
                    (LinearLayout.LayoutParams) child.getLayoutParams();
            float childExtra = lp.weight;
            if (childExtra > 0) {
                // 使用最大子视图高度进行测量
                // 如何才会出现这种测量方式，见分析五
                child.measure(
                        MeasureSpec.makeMeasureSpec(child.getMeasuredWidth(),
                                MeasureSpec.EXACTLY),
                        MeasureSpec.makeMeasureSpec(largestChildHeight,
                                MeasureSpec.EXACTLY));
            }
        }
    }
}
```
**分析四：delta 为负值时的情况？**
考虑如下布局
``` xml
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="300dp"
    android:background="@android:color/darker_gray"
    android:orientation="vertical">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout_weight="1"
        android:background="@android:color/holo_red_dark"
        android:text="TextView1"/>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout_weight="2"
        android:background="@android:color/holo_red_light"
        android:text="TextView2"/>
</LinearLayout>
```
预览效果如下：
![](/assets/images/LinearLayout onMeasure 过程分析/linearlayout_test01.png)
结论：TextView1 与 TextView2 的高度比为 2 : 1，即 TextView1 的高度为 200 dp，TextView2 的高度为 100 dp。
测量过程分析：
1. 第一次测量时，两个子视图都会进行测量。此时，TextView1 的高度 = TextView2 的高度 = 300 dp ，mToatalLength 为测量的子视图高度之和，即 600 dp。
2. 跳过重新计算 mTotalLength 阶段。
3. 重新测量子视图大小时，delta = -300 dp。
4. 重新分配剩余空间。TextView1 的高度为 300 + 1/3 * (-300 ) = 200 dp，同理可计算 TextView2 的高度为 100 dp。


**分析五：`measureWithLargestChild` 如何影响子视图高度的测量？**
根据分析三可知，如果 TextView1 设置了 `weight` 属性，它的高度就会被重新测量，测量高度为子视图最大高度，测量规格为 `EXACTLY` 。

> 由分析三和分析五可知：在高度测量规格不是 `Exactly` 的前提下，子视图不设置 `layout_weight` 属性，`LinearLayout` 设置 `measureWithLargestChild` 属性，仅仅会影响 `LinearLayout` 的高度测量。同时设置这两个属性，才会拉伸子视图至子视图最大高度。

另外一个问题，如果 TextView2 也设置了 `weight` 属性呢？
答案是对效果没有影响，但是在测量最后一步， `TextView2` 进行了一次多余测量。因为子视图最大高度由 `TextView2` 的高度决定，再次使用子视图最大高度对 `TextView2` 进行测量，测量结果与之前一致。

** 分析六：宽度测量过程 **
仅分析测量规格不为 `Exactly` 的情况。
- 如果子视图的布局参数均为 `match_parent` , `LinearLayout` 的宽度由子视图最大的宽度决定。
- 如果子视图的布局参数不全是 `match_parent` ，`LinearLayout` 宽度由非 `match_parent` 的子视图宽度决定。
