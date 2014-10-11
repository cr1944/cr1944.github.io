# 自定义控件：贝赛尔时钟（bezier-clock）

---

之前看到一个js实现的贝赛尔时钟效果[bezier-clock][1]，觉得不错，就想用java实现移植到android上，于是就有了这篇博文。

##贝赛尔曲线##

关于贝赛尔曲线`Bézier curve`，大家应该有所了解，网上也有很多文章，在此不详细解释，如果你还不知道原理，在继续往下看之前，先去维基百科[贝赛尔曲线][2]了解一下。

我们重点需要了解下三次曲线的绘制原理，由4个点确定一段曲线，见下图：
![三次曲线][3]

##为什么是三次曲线？##

原因很简单，因为我们要用到`Path`类的`cubicTo`方法来绘制曲线，该方法就是绘制三次贝赛尔曲线。首次调用`cubicTo`需要调用`moveTo`初始化起始点。

```java
public void draw(Canvas canvas, Path path, Paint paint) {
        path.reset();
        path.moveTo(vertexX, vertexY);
        path.cubicTo(controls[0][0], controls[0][4],
                controls[0][5], controls[0][6],
                controls[0][5], controls[0][5]);
        path.cubicTo(controls[1][0], controls[1][7],
                controls[1][8], controls[1][9],
                controls[1][6], controls[1][5]);
        path.cubicTo(controls[2][0], controls[2][10],
                controls[2][11], controls[2][12],
                controls[2][7], controls[2][5]);
        path.cubicTo(controls[3][0], controls[3][13],
                controls[3][14], controls[3][15],
                controls[3][8], controls[3][5]);
        ...
    }
```

每个数字用4条贝赛尔曲线即可实现，这里偷个懒，所有数字的顶点数据我都是利用前面提到的js实现的贝赛尔时钟里面的数据。

##动画##

接下来就是动画了，这里原理也很简单，通过一个浮点型的[0,1]因子，将当前数字的每个控制点的x，y坐标平滑转变到新数字的对应点的x，y坐标。

```java
public void setFactor(float f) {
        factor = f;
        vertexX = factor * (VERTEXX[newDigit] - VERTEXX[digit]) + VERTEXX[digit];
        vertexY = factor * (VERTEXY[newDigit] - VERTEXY[digit]) + VERTEXY[digit];
        for (int i = 0; i < 4; i++) {
            for (int j = 0; j < 6; j++) {
                controls[i][j] = factor * (CONTROLS[newDigit][i][j] - CONTROLS[digit][i][j]) + CONTROLS[digit][i][j];
            }
        }
        bezierClock.refresh(this);
    }
```
再跟当前时间的时分秒结合起来，一个贝赛尔时钟就出来了。

![此处输入图片的描述][4]

>**源代码**
[https://github.com/cr1944/bezier-clock][5]


  [1]: http://jackf.net/bezier-clock/
  [2]: http://zh.wikipedia.org/wiki/%E8%B2%9D%E8%8C%B2%E6%9B%B2%E7%B7%9A
  [3]: https://github.com/cr1944/bezier-clock/raw/master/3.gif
  [4]: https://github.com/cr1944/bezier-clock/raw/master/2.gif
  [5]: https://github.com/cr1944/bezier-clock
