---
layout: single
related: false
title:  Android 开发者选项的“指针位置”坐标值
date:   2020-06-10 23:52:00
categories: android
tags: display
toc: true
---

> 打开开发者选项中的“指针位置”，界面上方状态栏处会显示当前触屏的X/Y坐标，那么这个坐标值是怎么打印出来的呢？

<!--more-->

# 1. 代码分析

在frameworks/base/core/java/com/android/internal/widget/PointerLocationView.java的`onDraw`方法中，在触屏的时候会实时打印底层读取的X/Y值。

如下代码（Android Q AOSP源码）,`ps.mCoords.x`和`ps.mCoords.y`是底层传递读取的坐标值，float类型。

最后会显示成类似`X:500.5  Y:200.6`

```java
//PointerLocationView.java
    @Override
    protected void onDraw(Canvas canvas) {
        final int w = getWidth();
        final int itemW = w/7;
        final int base = -mTextMetrics.ascent+1;
        final int bottom = mHeaderBottom;

        final int NP = mPointers.size();

        // Labels
        if (mActivePointerId >= 0) {
            final PointerState ps = mPointers.get(mActivePointerId);

            canvas.drawRect(0, 0, itemW-1, bottom,mTextBackgroundPaint);
            canvas.drawText(mText.clear()
                    .append("P: ").append(mCurNumPointers)
                    .append(" / ").append(mMaxNumPointers)
                    .toString(), 1, base, mTextPaint);

            final int N = ps.mTraceCount;
            if ((mCurDown && ps.mCurDown) || N == 0) {
                canvas.drawRect(itemW, 0, (itemW * 2) - 1, bottom, mTextBackgroundPaint);
                canvas.drawText(mText.clear()
                        .append("X: ").append(ps.mCoords.x, 1)
                        .toString(), 1 + itemW, base, mTextPaint);
                canvas.drawRect(itemW * 2, 0, (itemW * 3) - 1, bottom, mTextBackgroundPaint);
                canvas.drawText(mText.clear()
                        .append("Y: ").append(ps.mCoords.y, 1)
                        .toString(), 1 + itemW * 2, base, mTextPaint);
            }
    ......
```

# 2. 问题案例

## 2.1. 问题描述

> 如果此时设备的分辨率是`1080x2340`，“指针位置”坐标值边缘滑动需要显示到`1079x2339`。而现在出现了问题：在竖屏的时候只能显示到`1078x2338`，横屏（两种横屏情况）只能显示到`1079x2338`和`1078xz2339`？此处如何进行修改？

## 2.2. 分析

首先要对此处读取的`ps.mCoords.x`和`ps.mCoords.y`值打印，发现在滑动到边缘的时候，应该显示1079，打印的值大约是`1078.0001`；应该显示2339的时候，打印的值大约是`2038.0001`。

所以在此处需要判断，在大于1078或2339的时候，使用`进一法`，将其作加一操作。

同时还要考虑到横屏和竖屏两种状态。

## 2.3. 修改

因为PointerLocationView.java继承view.java，可以使用`getResources().getConfiguration();`来获取设备参数，从而获取到当前横竖屏的状态。

```java
    //如果是true，则对坐标加一操作
    private boolean mRealX = false;
    private boolean mRealY = false;
    //获取当前设备横竖屏状态
    private int mOrientation;

    @Override
    protected void onDraw(Canvas canvas) {
        ......
                //获取横屏、竖屏状态并判断
                Configuration mConfiguration = getResources().getConfiguration();
                mOrientation = mConfiguration.orientation;
                if (mOrientation == mConfiguration.ORIENTATION_PORTRAIT) {
                  mRealX = ps.mCoords.x > 972.0f && ps.mCoords.x < 1079.0f;
                  mRealY = ps.mCoords.y > 2106.0f && ps.mCoords.y < 2339.0f;
                } else if (mOrientation == mConfiguration.ORIENTATION_LANDSCAPE) {
                  mRealX = ps.mCoords.x > 2106.0f && ps.mCoords.x < 2339.0f;
                  mRealY = ps.mCoords.y > 972.0f && ps.mCoords.y < 1079.0f;
                }

                //绘制的时候判断是否加一
                canvas.drawText(mText.clear()
                        .append("X: ").append(mRealX ? ps.mCoords.x + 1.0f : ps.mCoords.x, 1)
                        .toString(), 1 + itemW, base, mTextPaint);
                canvas.drawRect(itemW * 2, mHeaderPaddingTop, (itemW * 3) - 1, bottom,
                        mTextBackgroundPaint);
                canvas.drawText(mText.clear()
                        .append("Y: ").append(mRealY ? ps.mCoords.y + 1.0f : ps.mCoords.y, 1)
                        .toString(), 1 + itemW * 2, base, mTextPaint);
                mRealX = false;
                mRealY = false;
            } else {
    ......
```
