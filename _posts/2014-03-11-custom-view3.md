---
layout: post
title: 创建自定义View(3)
---

#Making the View Interactive#
绘制UI只是创建自定义控件的一部分。你还需要使你的控件模仿真实世界的情况响应用户输入。对象始终应该模仿真实对象的行为。例如，图片不应该突然消失然后重新出现在一个别的地方，因为真实世界的物体不会这样，图片应该从一个地方移动到另一个地方。 用户同样也能察觉到界面的微妙的行为，作出跟真实世界类似的反应。例如，当用户快速滑动一个界面时，他们应该能感觉到一开始的滞后感，以及结束时继续滑行一段的惯性。本节将演示如何使用Android framework的一些特性，给自定义控件添加这些真实世界的行为。 
##Handle Input Gestures##
跟其他UI框架一样，Android也有一套输入事件模型。用户的行为被转化成事件，并触发一些回调方法，你可以重写这些回调方法，来对用户的行为作出自定义的响应。Android系统最常见的事件是touch事件，它会触发`onTouchEvent(android.view.MotionEvent)`。重写这个方法来处理事件：
```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    return super.onTouchEvent(event);
}
```
Touch事件本身并没有什么特别的用处。现在的触控交互都是依据手势（如敲击，下拉，上推，飞行，缩放）来定义的。为了把原始的触摸事件转化成手势，Android提供了`GestureDetector`。 构造一个`GestureDetector`需要传递一个继承`GestureDetector.OnGestureListener`接口的类的对象作为参数。如果你只想处理部分手势，你可以只继承`GestureDetector.SimpleOnGestureListener`接口。例如：
```java
class mListener extends GestureDetector.SimpleOnGestureListener {
    @Override
    public boolean onDown(MotionEvent e) {
        return true;
    }
}
mDetector = new GestureDetector(PieChart.this.getContext(), new mListener());
```
无论是否使用`GestureDetector.SimpleOnGestureListener`接口，你都要实现`onDown()`方法并且返回`true`。这是因为，所有的手势都从`onDown()`消息开始的，如果你在`onDown()`中返回`false`，正如`GestureDetector.SimpleOnGestureListener`的做法，系统会以为你想要忽略其它的手势消息，那么 `GestureDetector.OnGestureListener`其它的方法永远不会被调用。你唯一需要在`onDown()`中返回`false`的情况就是你的确想要忽略所有的手势消息。一旦你实现了`GestureDetector.OnGestureListener`接口并且创建了 `GestureDetector`对象，你就可以使用你的`GestureDetector`对象来解析`onTouchEvent()`中接收到的触摸事件。
```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    boolean result = mDetector.onTouchEvent(event);
    if (!result) {
        if (event.getAction() == MotionEvent.ACTION_UP) {
            stopScrolling();
            result = true;
        }
    }
    return result;
}
```
当你传递给`onTouchEvent()`一个触摸事件，如果它不认为是手势操作的一部分，它就返回`false`。然后你就可以做一些特殊处理。 
##创建物理上的仿真运动##
手势是控制触摸屏设备的一个重要的方式，但是如果不仿照真实物理感受，它可能会变得违反常规，很难去适应。*飞行*手势就是一个很好的例子，用户快速地在屏幕上滑动一个手指，促使它滑行。这个手势这样才是合理的，界面先是快速移动，然后慢慢停下来，就好像用户推了一个滑轮让它滚动一样。 
然而，模拟滑轮的感觉不是那么简单。这里需要许多的物理学和数学知识来促使滑轮模型正确运作。幸运的是，Android提供了一些辅助类来模拟这样那样的行为。`Scroller`类是处理滑轮样式的*飞行*手势的基础。 要开始飞行，调用`fling()`，传入起始速率，最小和最大x，y坐标值。你可以通过`GestureDetector`来计算出速率。
```java
@Override
public boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX, float velocityY) {
    mScroller.fling(currentX, currentY, velocityX / SCALE, velocityY / SCALE, minX, minY, maxX, maxY); postInvalidate(); }
```
>注意：`GestureDetector`计算出的速率是物理精度的，许多开发者表示使用这个值的飞行动画太快了，通常的做法是把x，y方向的速率除以一个4到8因子。 调用`fling()`设置了飞行手势的物理模型，然后，你需要定期调用`Scroller.computeScrollOffset()`来更新`Scroller`。`computeScrollOffset()`通过读取当前时间并且使用物理模型获取该时间的x，y方向坐标值来更新`Scroller`的内部状态。调用`getCurrX()`和`getCurrY()`来获取这些值。大多数控件直接把`Scroller`对象的x，y值直接传递给`scrollTo()`方法。PieChart例子有些不同：它使用当前卷动的y值来设置图表的旋转角度。
```java
if (!mScroller.isFinished()) {
    mScroller.computeScrollOffset();
    setPieRotation(mScroller.getCurrY());
}
```
`Scroller`类为你计算滚动位置，但是它不会自动把这些位置作用到你的控件上。你的职责是要足够频繁地去获取和作用新的坐标位置，来使卷动动画看起来平滑。有两种方法：
>- 调用`fling()`之后调用`postInvalidate()`来强制重绘。这种方式要求你在`onDraw()`中计算卷动偏移量，并且在每次卷动偏移量改变时都调用`postInvalidate()`。
>- 设置一个`ValueAnimator`来对飞行时间进行动画处理，然后 通过调用`addUpdateListener()`添加一个listener来处理动画状态改变。 PieChart例子使用第二种处理。这种方式设置起来稍微复杂点，但是它运作起来更加接近动画系统，并且不需要潜在的不重要的view invalidate处理。缺点是API等级11前不支持`ValueAnimator`，所以这项技巧不能用于Android3.0前的设备。 

>注意：`ValueAnimator`在API等级11前不支持，但是你仍然可以在应用中使用它。你只需要保证在运行时检查API等级，如果低于11，忽略掉调用即可。
```java
mScroller = new Scroller(getContext(), null, true);
mScrollAnimator = ValueAnimator.ofFloat(0,1);
mScrollAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
    @Override
    public void onAnimationUpdate(ValueAnimator valueAnimator) {
        if (!mScroller.isFinished()) {
            mScroller.computeScrollOffset();
            setPieRotation(mScroller.getCurrY());
        } else {
            mScrollAnimator.cancel();
            onScrollFinished();
        }
    }
});
```
##平滑转变##
用户期望UI转换能够平滑，UI元素应该渐入渐出而不是突然出现和消失。运动平滑开始结束而不是突然开始结束。Android3.0引入的` property animation framework`，使得平滑转变更加容易。 要使用动画系统，当一个会影响控件的表现的属性变化了，不要直接去改变该属性值，使用`ValueAnimator`来改变属性值。下面的例子，改变PieChart中当前选中的切块，使得整个图表旋转，保证选中点始终在选中切块的中间。`ValueAnimator`在几百毫秒内渐渐改变旋转角度，而不是突然设置到新的角度。
```java
mAutoCenterAnimator = ObjectAnimator.ofInt(PieChart.this, "PieRotation", 0);
mAutoCenterAnimator.setIntValues(targetAngle);
mAutoCenterAnimator.setDuration(AUTOCENTER_ANIM_DURATION);
mAutoCenterAnimator.start();
```
如果你要改变的属性是View类的属性之一，动画就更容易了，因为控件有一个内部的针对多种属性同时进行动画的优化过的`ViewPropertyAnimator`，比如：
```java
animate().rotation(targetAngle).setDuration(ANIM_DURATION).start();
```
