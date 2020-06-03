---
date : 2018-11-08T14:06:26+08:00
tags : ["View", "Android"]
draft: true
title : "View 的一些整理 ZoomImageView"

---

# 写在最前面

截至到目前，我们对 View 原理、绘制都有一定的了解了。本篇我们就根据之前的学习经验来写一个可以手动放大缩小的 ImageView。截至到目前，我们对 Vi ew 原理、绘制都有一定的了解了。本篇我们就根据之前的学习经验来写一个可以手动放大缩小的 ImageView。截至到目前，我们对 View 原理、绘制都有一定的了解了。本篇我们就根据之前的学习经验来写一个可以手动放大缩小的 ImageView。截至到目前，我们对 View 原理、绘制都有一定的了解了。本篇我们就根据之前的学习经验来写一个可以手动放大缩小的 ImageView。截至到目前，我们对 View 原理、绘制都有一定的了解了。本篇我们就根据之前的学习经验来写一个可以手动放大缩小的 ImageView。截至到目前，我们对 View 原理、绘制都有一定的了解了。本篇我们就根据之前的学习经验来写一个可以手动放大缩小的 ImageView。截至到目前，我们对 View 原理、绘制都有一定的了解了。本篇我们就根据之前的学习经验来写一个可以手动放大缩小的 ImageView。


<!--more-->

## GestureDetector

```java
public interface OnGestureListener {
    //手指落下处罚，如果不返回 true 忽略之后的所有事件
    boolean onDown(MotionEvent e);
    //执行了down事件，还没执行up 或者 move 事件,在 longPress 之前执行
    void onShowPress(MotionEvent e);
    //抬起手指时执行 不能作为单击的事件判断
    boolean onSingleTapUp(MotionEvent e);
    //滑动时触发
    boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY);
    //长按触发
    void onLongPress(MotionEvent e);
    //惯性滑动
    boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX, float velocityY);
}

public interface OnDoubleTapListener {
    //单击动作确认
    boolean onSingleTapConfirmed(MotionEvent e);
    //双击确认（第二次点击的down事件）
    boolean onDoubleTap(MotionEvent e);
    //双击的所有动作（第二次down move cancel）
    boolean onDoubleTapEvent(MotionEvent e);
}
```