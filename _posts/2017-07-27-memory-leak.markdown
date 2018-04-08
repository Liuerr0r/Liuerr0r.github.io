---
layout:     post
title:      "Android内存泄漏学习笔记"
subtitle:   "学习总结"
date:       2017-07-27 12:00:00
author:     "溜大虾"
header-img: "img/leak/bg.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Android
    

---

# 内存泄漏

最近在项目中偶尔会发现内存泄漏现象。一开始还是一脸懵逼的查来查去，一直没有个清晰地思路。这几天闲下来，打算认真整理学习一下。我在这里从一个“如何主动造成内存泄漏”的角度来学习，然后熟悉一下不同方法检测的结果如何，这样以后再遇到相关问题时就能够很快的解决了。

## java gc

首先要有一个大前提，也就是java gc。在大部分虚拟机（包括Android的ART）中，Java都采用了“可达性分析”算法来进行内存回收，原理是：会有几个引用作为root节点，对于任意对象来说，如果从root层层遍历，如果找不到对于他的引用链，那么这个对象就被标记为无用，就会在gc时被销毁。

## 何为泄漏

内存泄漏，即部分对象虽然已经不再使用，但是因为有root持有引用，所以并没有被销毁，所占用的内存一直没有被释放。一次两次发生影响不大。如果频繁发生，那么可用内存会渐渐不足，最终在某一次请求内存时发现内存不足而发生oom。这里要明确一个概念，只有强引用会发生内存泄漏，而weak等引用因为其特殊机制，所以影响不大。

## 什么会泄露

泄露影响比较大的就是一些大对象，常见的比如某些资源，bitmap，以及activity。

## 如何发生泄露

首先让我们从另一个角度来看，如何主动发生内存泄漏呢？当然是想办法给他一个一直存在的强引用了。

### static

static这个关键字使一个变量变为只和这个类相关的类变量，和实例无关。他的生命周期是很长的，贯穿于app的启动到关闭。因此只要用一个static引用一个大对象，就可以泄漏了！举个例子：

```java
static Activity activity;
```

这是最简单粗暴的持有一个activity的引用，这样这个activity退出之后对象并没有被销毁。

```java
static View view;
```

一个View初始化时会用到context，我们在自定义View，重写构造方法时就知道这个了。因此如果一个View也像这样被持有，那个context也不会被释放。

### innerClass

内部类有个特性，是他会持有一个外部类的引用。如果内部类的实例一直存活，那么外部类activity的实例也就一直在。比如持有一个static的内部类引用：

```java
static LeakInnerClass context;

class LeakInnerClass {
    Context context;
}
```

或者以前我们用asynctask时喜欢搞一个匿名内部类执行异步任务，那当我们activity退出后这个异步任务还在执行的话，就会泄露了。

```java
void leakAsyncTask(){
    new AsyncTask<Void,Void,Void>(){

        @Override
        protected Void doInBackground(Void... params) {
            while(true){
              //哇啦啦啦啦啦啦我就是耗时操作
            }
            return null;
        }
    };
}
```

还有自己开个匿名线程：

```Java
void leakThread(){
    new Thread(){
        @Override
        public void run() {
            while (true){
                //哇啦啦啦啦啦啦我是耗时操作
            }
        }
    }.start();
}
```

还有在使用handler时，如果用了匿名handler，那么这个handler会带着activity的引用藏到消息队列中。消息没有被处理，就会造成内存泄漏。类似的，还有timertask等。

### register

我们平时会用到很多第三方库，比如ButterKnife EventBus RxJava等等，有的时候要获取系统服务，getSystemService。在使用的时候，都有一个先registerd或者bind的操作，而且在创建的时候会把activity的引用传过去。如果在activity结束时没有unregister或者unbind，就会造成内存泄漏。

## 如何检测泄漏

最简单的方法自然就是使用leakcanary了。只要给自己的项目加上这个工具，在发生泄漏的时候很快就会有提示。具体使用方法看[这里](https://www.liaohuqiu.net/cn/posts/leak-canary-read-me/)。

除此之外，android studio的刀耕火种的方式也不错，在这里我拿一个例子来示范一下我是怎么用的。

### 一次leak检测过程

#### 准备工作

首先，我写了两个activity，一个MainActivity，一个MemoryLeakActivity，逻辑是：MainActivity中有个按钮，点击会调到MemoryLeakActivity，在这个activity中会故意发生内存泄漏，代码如下：

![](/img/leak/code.jpg)  

在开始之前，再熟悉一下这个

![](/img/leak/moniter.jpg) 

（原谅我拙劣的画笔）

这个Monitors可以观察当前选中app的运行状态，现在只需要关注我标了123的地方。

首先这个Memory就是当前app的内存使用状况：

1.产生一个当前java堆的.hprof文件，这个文件反映了当前时刻java堆中内存详情，记住这个玩意有大用！

2.手动进行一次gc

3.这一块很重要，首先他有两个部分，蓝色和灰色。蓝色部分是当前内存使用大小，灰色部分是这个app被限制的最大内存大小。当蓝色部分越来越大，最后和灰色部分一样时，说明我们内存使用很多了即将内存不足，此时会进行一次gc同时将回灰色部分即限制的大小提高。

#### 肉眼观察

好了，介绍完这个工具，我们开始动手实践。首先打开app，点击按钮跳到会发生泄漏的activity上，再按返回键，然后再次按下按钮……这样反复操作：

![](/img/leak/activity1.jpg) 

![](/img/leak/activity2.jpg) 

与此同时，观察monitors的memory窗口，会发现蓝色部分在每一次开启新activity时会增长一部分，这很正常。但是在返回时，明明activity被“退出”了，但是蓝色部分还是没有变化。反复几次之后，蓝色部分一直在增长。也就是说当前内存越用越多，可以推断已经发生内存泄漏啦~

#### 自动分析

接下来由android studio来分析一下。在反复几次上面的操作之后，返回MainActivity，然后点击dump java heap按钮，然后等一会儿，android studio在为我们dump此时的horof文件。在成功后，会自动打开：

![](/img/leak/analyzer.jpg) 

如图在这个界面中，我们看最右面有一个栏叫 Analyzer Tasks，打开它，会发现有两个选项。我们是来看activity的内存泄漏的，那就把那个查重复字符串的√去掉。然后点右边那个绿色小三角，会发现下面Analysis Results栏里面展示出了当前泄露的Activity引用：

![](/img/leak/results.jpg) 

点击第一个item，最下方Reference Tree栏中便展示出了具体的引用：

![](/img/leak/inner.jpg) 

一般来说，第一个就是我们发生泄漏的地方。在图中，this$0的意思是隐式的引用。也就是说，我们的activity是因为一个内部类而发生了内存泄漏。

再点击刚才results中第二个item，看一下下方的reference tree:

![](/img/leak/ref.jpg) 

可以看到显式的有一个leakCntextRef引用，这说明我们有一个名为leakCntextRef的引用持有了activity。回过头看看我们的代码，果然，验证的没错。

#### 拓展

android studio的分析还算比较简单而且内容较少，我们可以把这个hprof导出，然后用mat来分析，具体看[这里](https://mingjunli.gitbooks.io/mat/content/)。



## 怎么解决泄漏

既然发生了泄漏，那就要解决它，避免问题出现。那么怎么解决呢？很简单，泄漏是因为持有了activity引用导致无法被销毁，那么只有两个选择：及时取消引用，或者让这个引用多待一会，但是该gc的时候就销毁。

根据这个思路：

- 我们在代码中能不用static变量持有contxt就不用，非要用就用weak引用。
- 对于内部类，尽量用静态内部类，这样就不会持有外部类引用。如果需要外部类引用做一些事，就手动赋给一个weak引用。
- 对于匿名内部类，不要图简单方便，实在不行就乖乖的写成外部类。
- 异步操作，尽量用可以方便管理的，比如rxJava，而不是用老古董AsyncTask了。非要用也最好加一个终止条件，在退出Activity时就该结束了。
- 在用rx时，可以在subscribe()的时候获取到Subscripeion，在不用的时候手动unSubscribe()，或者直接bind()到Activity的生命周期上，比如使用RxActivity管理。
- 在使用handler时，记得在activity的onDestroy()中加上remove()
- 在获取到某些资源时，使用完记得释放
- 在用到一些大对象比如Bitmap啊什么的，要记得回收
- 最后，在使用各种第三方库或者系统服务的时候还要记得有注册或绑定就要有解除注册、解绑定。