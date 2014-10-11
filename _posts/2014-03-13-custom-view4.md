---
layout: post
title: 创建自定义View(4)
---

##优化控件##
现在你已经有了一个设计的不错的控件，可以响应手势，在各种状态间切换，你还需要确保控件能够流畅运行。为了避免UI表现时卡顿或迟缓，你要确保你的动画始终保持60帧每秒运行。

<!--more-->

###少做频繁的处理###
要给控件提速，消除对一些代码的不需要的频繁调用。从`onDraw()`开始，它会给你极大的提升。你尤其应该避免在`onDraw()`中进行内存分配，因为有可能会导致内存回收（GC）而引发界面卡顿。在初始化的时候或者动画间隙分配对象，绝对不要在动画运行时分配对象。

除了给`onDraw()`瘦身之外，你还要注意，要尽可能不去频繁调用它。大部分`onDraw()`调用都是由于`invalidate()`调用触发的，所以，减少不必要的`invalidate()`。如果可以的话，尽可能调用四参数的`invalidate()`而不是无参数的`invalidate()`。无参数的`invalidate()`使整个控件刷新，而四参数的`invalidate()`只刷新指定区域。这种方式可以使绘制变得更加有效，并且避免一些不必要的超出区域的invalidate处理。

另一个代价昂贵的操作是遍历布局。控件每一次调用 `requestLayout()`，Android UI系统都需要遍历整个视图层次结构计算出每个控件需要的大小。一旦发现了冲突的测量，可能还需要进行多次遍历。UI设计者有时候使用嵌套的`ViewGroup`对象创建很深的层级结构以实现某个UI行为。这些深层次的结构会引发性能问题，尽可能使视图结构层次越浅越好。

如果你有一个复杂的UI，你应该考虑写一个自定义的`ViewGroup`来表现该布局。不同于内置的控件，你的自定义控件可以按照应用的需求设定子控件的大小和形状，也就避免了要靠遍历去计算子控件尺寸的计算。PieChart例子显示了如何继承一个`ViewGroup`作为自定义控件的一部分，PieChart有子控件，但它从来不去测量它们，而是根据它自己的布局算法直接设置它们的尺寸。

###使用硬件加速###
在Android 3.0,2D图形系统可以通过GPU加速。GPU硬件加速可以对大部分应用产生极大的性能提升。但它不是对每个应用来说都是合适的选择。framework可以让你精确地控制应用的哪一部分使用或者不使用硬件加速。

参考Android开发者指南的 Hardware Acceleration部分来获取关于如何在application，activity或者window层开启硬件加速。注意补充一点，你必须通过指定`AndroidManifest.xml`文件中`<uses-sdk android:targetSdkVersion="11"/>`来设置应用的target API到11或者更高。

一旦你开启了硬件加速，你可能也可能不会看见性能提升。GPU非常善于某些任务，如对位图图像缩放，旋转和平移。它们也不善于某些任务，如画线或曲线。为了最大程度地利用硬件加速，你应该最大化GPU善于处理的操作，最小化GPU不善于的操作。

在PieChart例子里，画饼图相当的代价昂贵，每次旋转的时候都去重绘使得UI变得迟钝。解决方式是把饼图放到一个子View中并且设置该子View的[layer type][1]为`LAYER_TYPE_HARDWARE`，因此GPU可以把它缓存为一个静态图片。例子中把子控件定义为`PieChart`的内部类，尽量最小化当要继承这项操作时要改变的代码量。

```java
private class PieView extends View {
    public PieView(Context context) {
        super(context);
        if (!isInEditMode()) {
            setLayerType(View.LAYER_TYPE_HARDWARE, null);
        }
    }
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        for (Item it : mData) {
            mPiePaint.setShader(it.mShader);
            canvas.drawArc(mBounds, 360 - it.mEndAngle, it.mEndAngle - it.mStartAngle, true, mPiePaint);
        }
    }
    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        mBounds = new RectF(0, 0, w, h);
}
    RectF mBounds;
}
```

这项更改之后，`PieChart.PieView.onDraw()`只有在控件第一次被显示的时候会调用，在其他的时间里，饼图都被缓存为图片，通过GPU在不同的旋转角度重绘。GPU加速尤其擅长这种处理，性能差异也是立竿见影。

然而这也需要一个权衡，缓存成硬件层图像消耗有限的显存空间。因此最终版本的`PieChart.PieView`只在用户滚动控件的时候设置为`LAYER_TYPE_HARDWARE`，在其他时间里都设置为 `LAYER_TYPE_NONE`，GPU将不会缓存图像。

最后，别忘了剖析你的代码。在一个控件上提升性能在另一个控件上可能就影响性能了。

  [1]: http://developer.android.com/reference/android/view/View.html#setLayerType%28int,%20android.graphics.Paint%29
