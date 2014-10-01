---
layout: post
title: 创建自定义View(1)
---

Android framework提供大量的控件，用于交互和各种数据的显示。但是有时候你的应用会有一些独特的需求，内置的控件无法满足。本节将会告诉你如何自己去创建一个强健的，可重用的控件。
 --------
##章节##
###创建一个View类###
 创建一个像内置控件一样的类，拥有自定义`attributes`，并且支持ADT布局编辑器。
###自定义绘制###
使用Android的图形系统独特呈现你的控件。
###Making the View Interactive###
用户希望一个控件对于手势操作可以表现的平滑自然，本节将会探讨如何进行手势识别，物理学以及动画，让你的用户体验体现得更加专业。
###优化View###
就算你的UI再漂亮，如果运行起来很卡，用户也不会喜欢它。学习如何避免常见的性能问题，如何使用硬件加速来使你的控件绘制运行得更快。

<!--more-->

 --------
##创建一个View类##
一个精心设计的控件就像一个精心设计的类一样：它把一系列的功能压缩进一个容易使用的用户接口中，它高效地使用CPU和内存，等等。相比较一个精心设计的类，一个精心设计的控件还需要： - 遵守Android标准 - 提供可用于自定义样式的`attributes`，用于Android XML布局 - 发送accessibility事件 - 兼容各种Android平台 Android framework提供一系列基本类和XML标签，帮助你创建一个满足这所有需求的控件。本节将探讨如何使用Android framework来创建一个控件的核心功能。
###继承View###
Android framework所有的控件类都继承自View，你的自定义控件也可以直接继承View，或者你可以继承某个已存在的控件以节约时间，如Button。 为了让ADT可以跟你的控件交互，你至少要提供一个构造方法，参数为`Context`和`AttributeSet`，该方法使布局编辑器可以创建和编辑你的控件的实例。

```java
class PieChart extends View {
    public PieChart(Context context, AttributeSet attrs) {
    }
} 
```

###定义客制化Attributes###
把一个内置的控件加到你的用户界面中，你可以在XML文件中加入对应的元素，并且通过元素属性去控制它的表现。一个写得很好的自定义控件也可以通过XML去添加和设计。你需要做的是：
* 在`<declare-styleable>`资源中定义`attributes`
* 在XML布局中指定`attributes`的值
* 运行时获取指定的`attribute`的值
* 把获取的`attribute`的值应用到你的控件上

本节将探讨如何定义`attributes`并指定它们的值。下节将探讨如何在运行时获取和使用它们。

要定义`attributes`，添加`<declare-styleable>`资源到你的工程，通常是把它们放在`res/values/attrs.xml`文件中，如下：

```xml
<resources>
    <declare-styleable name="PieChart">
        <attr name="showText" format="boolean" />
        <attr name="labelPosition" format="enum">
            <enum name="left" value="0"/>
            <enum name="right" value="1"/>
        </attr>
    </declare-styleable>
</resources>
```

 这段代码定义了两个自定义属性：`showText`和`labelPosition`，在一个名为`PieChart`的style中，这个style的名字按照惯例应该跟该自定义控件的名字一致。尽管这不是严格的要求，但是许多流行的编辑器依赖这个惯例来提供声明完成。

 一旦你定义了自定义`attributes`，你就可以在布局XML文件中像使用系统内置的`attributes`一样去使用它了。唯一的不同是，你的自定义`attributes`属于不同的命名空间`http://schemas.android.com/apk/res/[your package name]`，而不是`http://schemas.android.com/apk/res/android`。如：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:custom="http://schemas.android.com/apk/res/com.example.customviews"> 
    <com.example.customviews.charting.PieChart
        custom:showText="true"
        custom:labelPosition="left" />
</LinearLayout> 
```

 为了避免重复的命名空间URI，本例中使用`xmlns`指令，它把custom作为`http://schemas.android.com/apk/res/com.example.customviews`的别名，你也可以指定任意其它的别名。 注意添加自定义控件到布局中的XML标签的名字，它是自定义控件类的完全限定名。如果你的控件类是一个内部类，你必须把它的外部类的名字加上。如，`PieChart`类有个内部类`PieView`，要使用这个类的自定义attributes，你应该使用这个标签`com.example.customviews.charting.PieChart$PieView`。

###应用客制化Attributes###
若一个控件是从XML布局中创建的，所有XML标签中的`attributes`都是从资源bundle中读取并以`AttributeSet`传递到控件的构造方法中。尽管可以直接从`AttributeSet`中读取值，但是这种做法有些缺点：
- `attribute`的值中的资源引用无法处理
- `Styles`无法应用

正确的做法是，把`AttributeSet`传递到`obtainStyledAttributes()`中，该方法返回一个 `TypedArray`。

Android资源编译器会做很多事来使你调用 `obtainStyledAttributes()`更容易。对于res目录下的每一个`<declare-styleable>`， 生成的R.java都会定义一个attribute的id的数组和代表该数组中每一个attribute的索引的常量的集合。你可以使用这些预定义的常量去从`TypedArray`读取attributes。下面演示`PieChart`如何读取它的`attributes`：

```java
public PieChart(Context context, AttributeSet attrs) {
    super(context, attrs); TypedArray a = context.getTheme().obtainStyledAttributes( attrs, R.styleable.PieChart, 0, 0);
    try {
        a.recycle();
    }
} 
```

注意`TypedArray`对象是一个共享资源，用完之后必须被回收。

###添加属性和事件###
`Attributes`是一个控制控件的行为和表现的强大的方式，但是它们只能在控件初始化的时候才能被读取。为了提供动态的行为，需要考虑为每一个自定义`attribute`提供一个getter和setter方法。如：

```java
public boolean isShowText() {
    return mShowText;
}
public void setShowText(boolean showText) {
    mShowText = showText;
    invalidate();
    requestLayout();
}
```

注意`setShowText`方法调用了`invalidate()`和` requestLayout()`，这些调用是至关重要的。当任何可能会改变控件表现的属性变化了，你都要invalidate该控件，系统就会知道它需要重绘。同样的，当一个可能会影响控件的大小或形状的属性变化了，你需要进行一次重新布局。忘记这些方法调用可能会导致一些很难发现的bug。

###Design For Accessibility###
你的自定义控件应该考虑各种各样的用户。其中包括无法看见或者正常使用触摸屏的残障人士。为了支持这些残障人士，你应该：
- 使用android:contentDescription属性给你的输入框添加标签
- 在适当的时机通过`sendAccessibilityEvent()`发送`accessibility`事件
- 支持备用控制器，如方向键和轨迹球

（未完待续）
