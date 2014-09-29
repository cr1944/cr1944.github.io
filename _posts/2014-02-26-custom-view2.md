---
layout: post
title: 创建自定义View(2)
---

Android framework提供大量的控件，用于交互和各种数据的显示。但是有时候你的应用会有一些独特的需求，内置的控件无法满足。本节将会告诉你如何自己去创建一个强健的，可重用的控件。
 --------
#自定义绘制#
自定义控件最重要的是它的呈现，根据你的需求，自定义绘制可简可繁。本节涉及一些最常用的操作。
###重写onDraw()###
绘制一个自定义控件最重要的是重写`onDraw()`方法。它的参数是`Canvas`对象，控件用它来绘制它自己。`Canvas`类定义了方法用来绘制文本，线，位图以及其他图形单元。你可以使用这些方法在`onDraw()`中表现你的UI。在你绘制之前，你需要创建一个 `Paint`对象，下节讨论`Paint`更多细节。
###Create Drawing Objects###
`android.graphics`framework把绘制分成了两部分：
>- 画什么，通过`Canvas`处理
>- 怎么画，通过`Paint`处理

如，`Canvas`提供画线的方法，`Paint`提供方法来定义线的颜色；`Canvas`提供画矩形的方法，`Paint`提供方法来定义是实体颜色矩形还是空心矩形。简单的说，`Canvas`定义你可以画到屏幕上的形状，`Paint`定义它们的颜色，样式，字体等等。 因此，在你绘制前，你要创建一个或多个`Paint`对象。下面的`PieChart`例子演示通过`init`方法处理这些，该方法在构造方法中调用：
```java
private void init() {
    mTextPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    mTextPaint.setColor(mTextColor);
    if (mTextHeight == 0) {
        mTextHeight = mTextPaint.getTextSize();
    } else {
        mTextPaint.setTextSize(mTextHeight);
    }
    mPiePaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    mPiePaint.setStyle(Paint.Style.FILL);
    mPiePaint.setTextSize(mTextHeight);
    mShadowPaint = new Paint(0); mShadowPaint.setColor(0xff101010);
    mShadowPaint.setMaskFilter(new BlurMaskFilter(8, BlurMaskFilter.Blur.NORMAL));
    ... 
```
提前创建对象是一个重要的优化方式。控件重绘很频繁，并且许多绘制的对象有着昂贵的初始化过程。在`onDraw()`方法中创建这些绘制的对象，意味着性能下降，你的UI会表现的不尽人意。
###Handle Layout Events###
为了恰当地绘制你的控件，你需要知道它的尺寸是多少。复杂的自定义控件通常需要依据它们在屏幕上的尺寸和形状进行多重布局计算。绝对不要假定你的控件的尺寸。就算只有一个应用使用你的控件，该应用也需要处理不同的屏幕尺寸，不同的屏幕密度，以及在横竖屏模式下不同的宽高比。 尽管控件有许多方法来处理尺寸，它们大部分都不需要被重写。如果你的控件不需要对它的尺寸进行特别的控制，你只需要重写一个方法： `onSizeChanged()`。 `onSizeChanged()`在你的控件第一次被分配一个尺寸的时候会调用，当任何原因尺寸变化了它还会被调用。在`onSizeChanged()`中计算位置，尺寸，以及其它任何跟控件大小相关的属性，而不是在每次重绘的时候去计算。在`PieChart`的例子中，`onSizeChanged()`是计算边界矩形，文本标签的相对位置，以及其它视觉化元素的地方。 当你的控件被分配了一个尺寸，布局管理器假定该尺寸已包含了该控件的padding，在你计算尺寸的时候，必须处理该控件的padding。下面是`PieChart.onSizeChanged()`的一段代码：
```java
// Account for padding
float xpad = (float)(getPaddingLeft() + getPaddingRight());
float ypad = (float)(getPaddingTop() + getPaddingBottom());
// Account for the label
if (mShowText)
xpad += mTextWidth;
float ww = (float)w - xpad;
float hh = (float)h - ypad;
// Figure out how big we can make the pie.
float diameter = Math.min(ww, hh);
```
如果你需要对你的控件的布局参数有更精细的控制，实现 `onMeasure()`方法。该方法的参数是`View.MeasureSpec`，它告诉你该控件的父控件希望该控件有多大尺寸，以及该尺寸是一个硬性的最大值还是只是个建议值。作为一个优化方案，该值被存在一个封装的int型中，你可以用静态方法`View.MeasureSpec`来解析它。 下面是一个实现`onMeasure()`的例子。在此`PieChart`试图使它的区域足够大，来使它的饼图和标签一样大。
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
// Try for a width based on our minimum
int minw = getPaddingLeft() + getPaddingRight() + getSuggestedMinimumWidth();
int w = resolveSizeAndState(minw, widthMeasureSpec, 1);
// Whatever the width ends up being, ask for a height that would let the pie
// get as big as it can
int minh = MeasureSpec.getSize(w) - (int)mTextWidth + getPaddingBottom() + getPaddingTop();
int h = resolveSizeAndState(MeasureSpec.getSize(w) - (int)mTextWidth, heightMeasureSpec, 0); setMeasuredDimension(w, h);
}
```
这段代码有3个重要点：
>- 计算考虑到了控件的padding。正如之前所说，这是控件的责任。
>- 辅助方法`resolveSizeAndState()`用来得出最终的宽度和高度，该方法通过对比该控件想要的尺寸和传入`onMeasure()`的参数，返回一个恰当的`View.MeasureSpec`。
>- `onMeasure()`没有返回值。该方法通过`setMeasuredDimension()`传递结果。该方法是强制调用的，如果你遗漏了，该控件会报出一个运行时异常。

#Draw!#
一旦你完成了对象创建和测量的代码，你就可以实现`onDraw()`方法了。每一个控件实现`onDraw()`的方式都不一样，但是它们总会共享一些常用的操作：
>- 用`drawText()`绘制文本。 调用`setTypeface()`设置字体，`setColor()`设置文字颜色。
>- 用`drawRect()`，`drawOval()`和`drawArc()`绘制基本形状。调用`setStyle()`来设置是否填充或者轮廓或二者皆有。
>- 用`Path`类来绘制更复杂的形状。通过添加线和曲线到一个`Path`对象来定义一个形状，然后使用`drawPath()`来绘制这个形状。跟基本形状一样，`Path`也可以通过`setStyle()`来设置是否填充或者轮廓或二者皆有。
>- 通过创建`LinearGradient`对象来定义一个渐变。对`LinearGradient`对象调用`setShader()`来填充一个形状。
>- 使用`drawBitmap()`来绘制位图。 

举个例子，下面是绘制`PieChart`的代码，它混合使用了文本，线和形状：
```java
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    // Draw the shadow
    canvas.drawOval( mShadowBounds, mShadowPaint );
    // Draw the label text
    canvas.drawText(mData.get(mCurrentItem).mLabel, mTextX, mTextY, mTextPaint);
    // Draw the pie slices
    for (int i = 0; i < mData.size(); ++i) {
        Item it = mData.get(i);
        mPiePaint.setShader(it.mShader);
        canvas.drawArc(mBounds, 360 - it.mEndAngle, it.mEndAngle - it.mStartAngle, true, mPiePaint);
    }
    // Draw the pointer
    canvas.drawLine(mTextX, mPointerY, mPointerX, mPointerY, mTextPaint);
    canvas.drawCircle(mPointerX, mPointerY, mPointerSize, mTextPaint);
} 
```
