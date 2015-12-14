#Android Ptr Comparison
Only compare star > 1500 repos in github.

##Repo
|Repo|Owner|Star<br/>(2015.12.5)|version|Snap shot|
|:--:|:--:|:------:|:---:|:--:|
|[Android-PullToRefresh][3]<br/>(Author deprecated)|[chrisbanes][4]|6014|latest|![chrisbanes](/demo_gif/chrisbanes.gif)|
|[android-Ultra-Pull-To-Refresh][1]|[liaohuqiu][2]|3413|1.0.11|![liaohuqiu](/demo_gif/liaohuqiu.gif)|
|[android-pulltorefresh][5]<br/>(Author deprecated)|[johannilsson][6]|2414|latest|![johannilsson](/demo_gif/johan.gif)|
|[Phoenix][7]|[Yalantis][8]|1897|1.2.3|![yalantis](/demo_gif/yalantis.gif)|
|[FlyRefresh][9]|[race604][10]|1843|2.0.0|![flyrefresh](/demo_gif/flyrefresh.gif)|

##Expansion

|Repo|Custom header view|Support content layout|
|:--:|:--:|:--:|:---:|:--:|:--:|
|[Android-PullToRefresh][3]|<br/>由于仅支持其中实现的`LoadingLayout`作为顶视图，改代码实现自定义工作量较大。|任意视图，内置:`GridView`<br/>`ListView`,`HorizontalScrollView`<br/> `ScrollView` ,`WebView`|
|[android-Ultra-Pull-To-Refresh][1]|任意视图。<br/> 通过继承`PtrUIHandler`并调用<br/>`PtrFrameLayout.addPtrUIHandler()`得到最大支持。|任意视图|
|[android-pulltorefresh][5]|不支持，只能改代码。<br/> 代码仅一个`ListView`，耦合度太高，改动工作量较大。|无法扩展，自身为`ListView`|
|[Phoenix][7]|不支持，此控件特点就是顶部视图及动画。|任意视图|
|[FlyRefresh][9]|不支持，此控件特点就是顶部视图及动画。|任意视图|

##Easy use

|Repo|use in gradle|Pull up(Load more)|Auto load|Move resistance|
|:--:|:--:|:------:|:---:|:--:|:--:|:--:|
|[Android-PullToRefresh][3]|×|√|×|UI translation/move-1/2|
|[android-Ultra-Pull-To-Refresh][1]|√|×|√|√|
|[android-pulltorefresh][5]|×|×|×|UI translation/move-1/1.7|
|[Phoenix][7]|√|×|×|×|
|[FlyRefresh][9]|√|×|×|×|

##Performance

通过捕捉如下图中的操作持续1秒钟的systrace进行性能分析：

![trace_operation](trace_operation.gif)

> 注：由于开源库Header大多无法直接放自定义视图，头部视图复杂程度不同，数据对比结果会有所偏差。

###1. Chris Banes's Ptr

滑动实现方式：`View.post(Runnable)` + `View.scrollTo()` 

trace snapshot:

![trace_chrisbanes](/traces/chrisbanes.PNG)

作为Github上星星数最多的Android下拉刷新控件，从性能上看（渲染时间构成）几乎没有什么明显的缺点。可惜的是作者已经不再维护，并且gradle中也无法使用。在本次demo这类层级比较简单的环境中，几乎都达到了60fps，可以与后面的trace对比。

###2. liaohuqiu's Ptr

滑动实现方式：`Scroller` + `View.post(Runnable)` + `View.offsetTopAndBottom()`

trace snapshot:

![trace_liaohuqiu](/traces/liaohuqiu.PNG)

这套开源库可以说是自定义功能最强的组件了，美中不足的就是在下拉状态变化的时候会有一阵measure时间。我查看了一下代码，发现是`PtrClassicFrameLayout`的顶部视图出了问题：

![liaohuqiu_header](/liaohuqiu_ptr_header.PNG)

看！都是`wrap_content`，那么当里面的内容变化的时候，是会触发`View.requestLayout()`的。不要小看这一个子视图的小操作，一个`requestLayout()`大概是这么一个流程：`View.requestLayout()`->`ViewParent.requestLayout()`->...->`ViewRootImpl.requestLayout()`->`ViewRootImpl.doTraversal()`=>**MEASURE**(ViewGroup)=>**MEASURE**(ChildView of ViewGroup)

在层级复杂的时候（大部分互联网产品由于复杂的产品需求嵌套都会比较多），它会层层向上调用，将measure时间放大至一个可观的层级。下拉刷新界面的卡顿由此而来。

我修改了一下，将其全部变为固定高度、宽度，之后的trace如下：

![trace_liaohuqiu_new](/traces/liaohuqiu_new.PNG)

measure时间神奇的没掉了吧:P

###3. johannilsson's Ptr

滑动实现方式：`View.setPadding()`

trace snapshot:

![trace_johan](/traces/johan.PNG)

分析：

通过顶视图调用`View.setPadding()`来实现的滑动，是会造成不断的`requestLayout()`!这就解释了为什么图中UI线程的蓝色块时间(measure时间)很明显。**当你在视图层级比较复杂的app中使用它时，下拉动作所造成的开销会非常明显，卡顿是必然结果。**

###4. Yalantis's Ptr

滑动实现方式：`Animation` + `View.topAndBottomOffset()`

顶部动效实现方式：`Drawable`的`Canvas`中设置偏移量及缩放。

trace snapshot:

![trace_yalantis](/traces/yalantis.PNG)

分析：此开源库动画效果非常柔和，且顶部视图全部是通过draw去更新，不会造成第三个开源库那样的大开销问题。可惜的是比较难以去自定义顶部视图，不好在大型线上产品中使用，不过这个开源库是一个好的练手与学习的对象。由于顶部动效实现开销不大，它的性能同样非常好。

###5. race604's Ptr

滑动实现方式：`View.topAndBottomOffset()`

顶部动效实现方式：

- **飞机滑动** `ObjectAnimator`.
- **背景缩放** 通过放大系数计算`Path`后进行draw.

trace snapshot:

![trace_flyrefresh](/traces/flyrefresh.PNG)

分析：每次拖动都会重新计算背景"山体"与"树木"的`Path`，造成了draw时间过长。效果不错，也是一个好的学习对象，相比`Yalantis`的下拉刷新性能上就差一些了，它的draw中计算量太多。使用起来疑似bug。

##附录-知识点参考

1. [为你的应用加速 - 安卓优化指南](https://github.com/bboyfeiyu/android-tech-frontier/blob/master/issue-27/%E4%B8%BA%E4%BD%A0%E7%9A%84%E5%BA%94%E7%94%A8%E5%8A%A0%E9%80%9F%20-%20%E5%AE%89%E5%8D%93%E4%BC%98%E5%8C%96%E6%8C%87%E5%8D%97.md)
2. [使用Systrace分析UI性能](https://github.com/bboyfeiyu/android-tech-frontier/blob/b7e3f1715158fb9f2bbb0f349c4ec3da3db81342/issue-26/%E4%BD%BF%E7%94%A8Systrace%E5%88%86%E6%9E%90UI%E6%80%A7%E8%83%BD.md)

[4]: https://github.com/chrisbanes
[3]: https://github.com/chrisbanes/Android-PullToRefresh
[2]: https://github.com/liaohuqiu
[1]: https://github.com/liaohuqiu/android-Ultra-Pull-To-Refresh
[5]: https://github.com/johannilsson/android-pulltorefresh
[6]: https://github.com/johannilsson
[7]: https://github.com/Yalantis/Phoenix
[8]: https://github.com/Yalantis
[9]: https://github.com/race604/FlyRefresh
[10]: https://github.com/race604