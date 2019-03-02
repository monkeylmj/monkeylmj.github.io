---
title: "路径动画的实现方案"
date: 2018-04-08 20:10:38 +0800
tags: 
 - UI
---

总结一下工作中遇到的一个“UI路径动画”的实现。效果如下图（对号的动画）

<!-- more -->

![path tracing](https://ws2.sinaimg.cn/large/006tKfTcgy1g0o7yykaxcg30a00a0tei.gif)



## 基本分析

我们都知道，使用Android的Path Api可以很容易地画出一个对号（或者任意其他的不规则图形），实现这个效果的难点在于怎么以动画的形式逐渐地把这个路径给绘制出来。这里把路径(Path)从0到1绘制出来的过程称之为路径追踪（Path Tracing).

[@Path的绘制API]

```java
//构造任意一个path
Path path = new Path();
path.moveTo(0, 0);
path.lineTo(100,100);
path.lineTo(100, 200);
//画出path
cavas.drawPath(path, paint);
```



## 实现方案一（PathMeasure)

Android提供了PathMeasure类，用来进行路径的计算，可以用来实现路径追踪。

创建PathMeasure：

```java
PathMeasure pathMeasure = new PathMeasure(Path path, boolean forceClosed);
```

或者也可以使用无参构造函数:

```java
PathMeasure pathMeasure = new PathMeasure();
pathMeasure.setPath(Path path, boolean forceClosed);
```

第一个参数传入需要进行计算的路径，第二个参数指示是否将路径闭合。



几个有用的API

1. 获取路径的像素长度：

```java
pathMeasure.getLength();
```

2. 获取路径的片段：

```java
pathMeasure.getSegment(float startD, float stopD, Path dst, boolean startWithMoveTo)
```

这个方法是绘制路径追踪动画的核心方法，通过这个方法可以获取整个Path你想要的片段，**保持起点不变，终点一点点变长**，就可以将路径一点一点地绘制出来了。

*参数介绍*：

* startD和stopD：开始距离和结束距离，控制截取的Path的内容。
* dst:将截取的路径保存到这个Path中，之后绘制这个Path即可。
* startWithMoveTo: 通常为true，表示对dst path先调用moveTo移动到起点的位置。

相当于

```java
dst.moveTo(起点);
dst.lineTo(终点);
```

如果设为false，经过测试，相当于:

```java
dst.lineTo(起点);
dst.lineTo(终点);
```

一个Path如果直接lineTo某个点，隐式地在最开始添加了一个moveTo(0,0)，这种情况下获取的Path实际上起点连接到了(0,0)点。



使用上面两个API，结合属性动画，实现路径追踪的效果：

```java
public class RouteTraceView extends View {

    private Path mDstPath = new Path();
    private float mEndDistance;
    private PathMeasure mPathMeasure;

    private Paint mPaint;

    public RouteTraceView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    private void init() {
        mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        mPaint.setColor(Color.RED);
        mPaint.setStyle(Paint.Style.STROKE);
        mPaint.setStrokeWidth(10);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        mDstPath.reset();
        mDstPath.lineTo(0, 0);
        mPathMeasure.getSegment(0, mEndDistance, mDstPath, false);
        canvas.drawPath(mDstPath, mPaint);
    }

    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);

        //要动态绘制出来的路径
        Path path = new Path();
        path.moveTo(0, 50);
        path.lineTo(100, 100);
        path.lineTo(100, 200);

        mPathMeasure = new PathMeasure();
        mPathMeasure.setPath(path, false);
        final float length = mPathMeasure.getLength();

        ValueAnimator valueAnimator = ValueAnimator.ofFloat(0, 1);
        valueAnimator.setDuration(3000);
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                //动画的形式改变终点的距离
                mEndDistance = (float) animation.getAnimatedValue() * length;
                invalidate();
            }
        });
        valueAnimator.start();
    }
}
```



是不是很简单，对于任意一个Path，都能通过这种改变终点距离的方式，一点点地追踪出来~

## 实现方案二（DashPathEffect）

这种方式比较黑科技，算是奇技淫巧，用起来也是十分方便。

Paint画笔有这样一个方法:

```java
mPaint.setPathEffect(PathEffect effect);
```

用来给画出的路径添加bling bling的特效。



### PathEffect的实现类有以下几个：

* CornerPathEffect

![](https://i.loli.net/2019/03/02/5c79ecb89dd62.jpg)

* DecretePathEffect  随机偏移的乱七八糟的效果

![](https://i.loli.net/2019/03/02/5c79ece126611.jpg)

* DashPathEffect

![](https://i.loli.net/2019/03/02/5c79ececdb632.jpg)

* PathDashPathEffect  用某个Path来绘制虚线

![](https://i.loli.net/2019/03/02/5c79ecf8ec9e2.jpg)

* SumPathEffect  组合两种Effect，分别按照两种PathEffect对路径进行绘制

![](https://i.loli.net/2019/03/02/5c79ed02256d4.jpg)

* ComposePathEffect 组合两种Effect，先对目标路径应用第一个Effect，对改变后的Path再应用第二个Effect

  ![](https://i.loli.net/2019/03/02/5c79ed0b3da1b.jpg)




那这些PathEffect跟要实现的路径追踪有半毛钱关系吗？？？

too young too simple！看我是怎么用DashPathEffect来装逼的！

### 看一下DashPathEffect的构造方法：

[@DashPathEffect构造方法]

```java
public DashPathEffect(float intervals[], float phase){}
```

* intervals是一个数组，它的大小必须为偶数，最少两个。其中的元素按照【画线长度，空白长度，画线长度，空白长度….】排列。

比如：

```java
PathEffect pathEffect = new DashPathEffect(new float[]{20, 10, 5, 10}, 0); 
```

就表示在路径上先画20px的线，再画10px的空白，再画5px的线，再画10px的空白，再循环以此类推。

* phase表示相位的偏移，经过测试**phase为正时，虚线整体向起点方向偏移**，比如上述代码中offset如果为20,则我们先看到的则是10px的空白了~




### 黑科技马上要来了！

思考下面这种写法：

```java
//要动态绘制出来的路径
Path path = new Path();
path.moveTo(0, 50);
path.lineTo(100, 100);
path.lineTo(100, 200);
//拿到Path的长度
PathMeasure pathMeasure = new PathMeasure();
pathMeasure.setPath(path, false);
float pathLength = pathMeasure.getLength();
//特效
DashPathEffect dashPathEffect = new DashPathEffect(new float[]{pathLength, pathLength}, 0);
mPaint.setPathEffect(dashPathEffect);
canvas.drawPath(path, mPaint);
```

其中intervals数组画线的长度和空白的长度都是path的长度。按照前面讲的，这样整个路径画出来，先是实线部分，就把路径填满了，这个路径一下子全画了出来。

稍微修改一下，将phase改为pathLength：

```java
//特效
DashPathEffect dashPathEffect = new DashPathEffect(new float[]{pathLength, pathLength}, pathLength);
```

按照前面讲的，虚线整体向起点移动pathLength个像素，这样实线部分完全移出了路径，隐藏起来了，显示的完全是空白的部分（长度也是pathLength）。

到这里，完全隐藏和完全显示两个状态都有了，剩下的中间状态，通过一个属性动画将phase从pathLength变到0，整个路径就慢慢地显示出来了！！!

## 参考资料

* http://www.curious-creature.com/2013/12/21/android-recipe-4-path-tracing/
* https://www.jianshu.com/p/81150d4740a4
* http://hencoder.com/ui-1-2/