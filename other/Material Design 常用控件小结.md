---
date : 2018-09-13T16:25:34+08:00
tags : ["Material Design", "Android", "用法"]
title : "Material Design 常用控件小结"
draft : true
---

### 写在最前面
上一篇中，我们说到了 Behavior ，那就不能不提及 Material Design 。Android 5.0 引入的设计理念，当然在国内大家还是各自一套设计理念，我个人是挺喜欢 MD 的设计理念，直观，简约。本篇就来聊一下常用的控件。

### AppBarLayout
我们直接从 AppBarLayout 开始，其常和 ActionBar 连用，主要是在页面滑动时，对于标题栏做隐藏显示的操作。AppBarLayout 是一个`垂直的LinearLayout`，为其子控件提供了 ScrollFlags 属性，根据这个属性，如果页面有滑动，其便会有不同的滑动效果。
下面，我们就来看看，有哪些标志位可供使用。

    scroll  AppBarLayout 会跟随滚动事件一起发生移动。好像其本身也是属于 ScrollView 一样。
    enterAlways 当ScrollView往下滚动时，该View会直接往下滚动。而不用考虑ScrollView是否在滚动。要和scroll连用。
    exitUntilCollapsed 当这个View要往上逐渐“消逝”时，会一直往上滑动，直到剩下的的高度达到它的最小高度后，再响应ScrollView的内部滑动事件。要和scroll一起使用。
    enterAlwaysCollapsed 类似于exitUntilCollapsed，View在往下“出现”的时候，首先是enterAlways效果，当View的高度达到最小高度时，View就暂时不去往下滚动，直到ScrollView滑动到顶部不再滑动时，View再继续往下滑动，直到滑到View的顶部结束。要和scroll，enterAlwaysCollapsed一起使用。
    snap 简单理解，就是Child View滚动比例的一个吸附效果。也就是说，Child View不会存在局部显示的情况，滚动Child View的部分高度，当我们松开手指时，Child View要么向上全部滚出屏幕，要么向下全部滚进屏幕，有点类似ViewPager的左右滑动。

注意：如果需要一个可滑动显隐的 AppBarLayout，scroll 标志位一定要有。其次enterAlwaysCollapsed exitUntilCollapsed 两个不能同时使用，由于 exitUntilCollapsed 会使收起的 AppBarLayout 保留一定高度，同时 enterAlwaysCollapsed 也会让下拉的 AppBarLayout 显示出一定的高度，后者会在前者的基础上加，因此会发现 AppBarLayout 两种状态下高度不同。因此常用的组合有两种：`scroll|exitUntilCollapsed` `scroll|enterAlwaysCollapsed|enterAlways` 。第一种，上划不会完全收起，第二种，上划完全收起，下来不会完全展开

最后就是如果使用了这些标志位，记得在 ScrollView 上增加 `app:layout_behavior="@string/appbar_scrolling_view_behavior"` 这样 AppBarLayout 才知道以哪个ScrollView 作为参照进行显隐折叠。

### CollapsingToolbarLayout
CollapsingToolbarLayout 实现了一个折叠的 App bar，本身是一个 FrameLayout，因此内部元素无法排序，通常用作 AppBarLayout 的直接子控件。常用属性有一下这些：

    title 设置 CollapsingToolbarLayout 上的标题内容，缺省的话使用包括的 Toolbar 的标题
    titleEnabled ture 则显示大标题，否则只显示 App Bar 小标题（如果有的话）。
    contentScrim 当收起的时候，显示的 App Bar 的颜色。
    scrimAnimationDuration 设置颜色渐变的长度。
    expandedTitleGravity 设置展开时 title 的位置，可用 | 隔开多个位置。

还有`layout_collapseMode` 属性，由于 CollapsingToolbarLayout 内部可能有多种元素，所以其各个元素的表现也可能是不同的，因此这里也只有两种标志位，分别是： 

    pin：有该标志位的View在页面滚动的过程中会一直停留在顶部，比如Toolbar可以被固定在顶部 
    parallax：有该标志位的View表示能和页面同时滚动。与该标志位相关联的一个属性是：layout_collapseParallaxMultiplier，该属性是视差因子，表示该View与页面的滚动速度存在差值，造成一种相对滚动的效果。

注意如果 CollapsingToolbarLayout 中含有一个 Toolbar 则最小高度为 Toolbar 的高度不用重新设置，主要是结合 scroll_flags 使用。

### NavigationView
导航栏继承于 Fragment ，因此内部放置的控件也是无法排序的。通常在使用的时候会设定

    app:menu="@menu/menu_drawer"
    app:headerLayout="@layout/nav_head_main"
这两字段，使得其设定 header 和 选项。当然也可以不设定这两个字段，直接在 NV 内部放置需要的控件。