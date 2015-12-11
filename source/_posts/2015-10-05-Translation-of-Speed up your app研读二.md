---
layout: default
title: Speed up your app 研读（二）
comments: true
---

本文为Speed up your app的研读第二部分
原文请参照 http://blog.udinic.com/2015/09/15/speed-up-your-app。

###一、GPU Profiling

Android Studio 1.4中新增的功能，主要用于显示GPU的解析

![gpu-profiling-1](/assets/images/speed-up-your-app/gpu-profiling-1.png)

该图表明了一个frame被解析的过程，颜色代表了解析的不同阶段。

1.蓝色代表draw：这个部分建立/更新了展示列表对象（display list），这些展示对象随后会被解析成GPU可以理解的OpenGL命令。如果蓝色所占区域较大则可能是因为复杂的视图，需要更多的时间去建立他们的展示列表（display list）。

2.紫色代表prepare：在android5.0版本中，增加了一个帮助ui线程更快解析ui的线程，他叫做RenderThread，该线程就负责上面第一点提到的工作，将display list 转换为OpenGL命令，并且把它传递到GPU中。当renderThread负责该方面的工作时，ui线程就可以处理下一个frame的工作。而ui线程将所有这些解析所需要的资源传递给renderThread所花费的时间就是紫色部分的时间。如果我们有很多资源需要传递给renderThread，那么势必会花费很多时间。

3.红色代表process：表示将第一步得到的display list，进行解析，将其转换为OpenGL命令。

4.黄色代表execute：将得到的OpenGL命令传递给GPU。这个部分由于cpu会附带传递buffer给GPU，而且会期望为了下一个frame得到干净的buffer而成为阻塞进程。并且由于buffer的总数是被限制的。如果GPU很繁忙的话。、

还有一些其他的阶段，这里就不再介绍，详情可以看原文

当然，如果我们要使用这个功能，我们首先得开启这个功能。

![gpu-profiling-2](/assets/images/speed-up-your-app/gpu-profiling-2.png)

或者使用命令行

```
adb shell dumpsys gfxinfo <PACKAGE_NAME>
```

当然，我们可以选择on screen as bars 这个选项。

![gpu-profiling-3](/assets/images/speed-up-your-app/gpu-profiling-3.png)

注意观察其中的绿线，超过了绿线就代表超过了16ms，出了问题。

###二、Hierarchy Viewer

![Hierarchy-Viewer-1](/assets/images/speed-up-your-app/Hierarchy-Viewer-1.png)

这个view hierarchy 在同一行，代表后面的元素是前面元素的子类，而同一列则他们都是同一个父元素的并列子元素。如果高度过于高，比如大于10层，则会在layout和measurement阶段花费我们更多的时间。

同样注意这里的颜色反映的问题。

![Hierarchy-Viewer-2](/assets/images/speed-up-your-app/Hierarchy-Viewer-2.png)

###三、Overdraw

Overdraw一般发生在你在某个界面上再画某些东西，比如你在红色背景上再添加一个黄色按钮，那么黄色按钮就是在红色背景上重画一次。Overdraw的存在是正常的，但是为了使我们的APP性能更好，我们要避免一些不必要的OverDraw。重画一次两次是没有关系的，甚至某些小区域重画四次也没有错，但是如果我们在屏幕上看到大部分区域都是红色，那么我们就是遇到了问题。


![overdraw-1](/assets/images/speed-up-your-app/overdraw-1.png)

![overdraw-2](/assets/images/speed-up-your-app/overdraw-2.png)

看右边的这个例子，这种overdraw往往是因为我们给activity或者fragment设置了默认的全屏背景色。比如默认的主题就会给我们的屏幕设置默认背景色。如果我们的activity有一个默认的覆盖整个屏幕的透明layout，那么我们应该去除掉window的背景色。在onCreate中使用


```
getWindow().setBackgroundDrawable(null);

```

当然我们可以和之前提到的Hierarchy Viewer一起解决这个问题。

###四、Alpha

setAlpha方法带来的性能问题
比如如下图，我们有三个imageview，他们交叠在一起
首先如果我们对这三个图片分别使用setAlpha 那么就会出现下面的情况

![alpha-1](/assets/images/speed-up-your-app/alpha-1.png)

这并不是我们想要的情况，幸运的是操作系统会帮助我们自动进行如下的转换，这种转换使用到了off-screen缓冲，并且会用到一个不会被检测到overdraw层。同时，由于系统不知道什么时候进行这种转换，所以他会对所有的情况都使用这种性能较差的情况。换句话说就是性能会很差。

![alpha-2](/assets/images/speed-up-your-app/alpha-2.png)

那么如何避免呢
1.Textview 使用setTextColor而不是setAlpha

2.ImageView使用setImageAlpha而不是setAlpha

3.自定义view，如果不支持交叠view，则可以不使用如下方法，但我们可以覆盖hasOverlappingRendering方法并返回false做到提示系统不使用复杂的处理方式。或者我们也可以覆盖onSetAlpha并返回true来告诉系统，当系统设定alpha的时候自动调用第一种方法。

###五、Hardware Acceleration

硬件加速有两方面好处，第一方面，它引入了displaylist 结构，用来记录视图绘制所需要的命令，用来更快的解析视图。第二方面就是很多人忽略的，view layers。使用view layers，我们可以解析view至off-screen缓存。这个特性主要用于动画。如果我们不使用这个特性，那么视图就会在每一次应用一个动画的时候进行重绘。在重绘时，也会将重绘操作传递给他的子类，这对于一个复杂的视图，是什么耗时的。

Using a View layer, we can render the View into an off-screen buffer (as seen earlier, when applying an Alpha channel) and manipulate it as we like. This feature is mainly great for animations, because we can animate complex Views quicker. Without layers, animating a View will invalidate it after changing the animated property (e.g. x coordinate, scale, alpha value etc.). For complex views, this invalidation propagates to all the child views, and they in turn will redraw themselves, a costly operation. Using a View layer, backed by Hardware, a texture is created in the GPU for our view. There are several operations we can apply on that texture without invalidating it, such as x/y position, rotation, alpha and more. All that means, that we can animate a complex view on our screen without invalidating it at all during the animation! This will make the animation much smoother. Here’s a code example how to do this

硬件加速代码十分简单

```
// Using the Object animator
view.setLayerType(View.LAYER_TYPE_HARDWARE, null);
ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(view, View.TRANSLATION_X, 20f);
objectAnimator.addListener(new AnimatorListenerAdapter() {
    @Override
    public void onAnimationEnd(Animator animation) {
        view.setLayerType(View.LAYER_TYPE_NONE, null);
    }
});
objectAnimator.start();

// Using the Property animator
view.animate().translationX(20f).withLayer().start();
```

但是在使用硬件加速时，我们需要注意两点
1.Clean up after your view，在动画结束以后，将硬件加速占有的内存清除掉，这里可以直接调用withlayer函数。他会在动画结束时帮我们自动清除硬件加速所占有的内存。

2.同时需要注意如果你使用的是对硬件加速并不支持的动画就会导致硬件加速实际上是失效的。
