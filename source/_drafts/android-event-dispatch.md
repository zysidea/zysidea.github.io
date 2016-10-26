---
title: Android事件分发机制总结
date: 2016-10-26 10:43:06
categories: 
  - Android
tags: 
---
一直一来对于**Android的事件分发机制**不够清晰，在自定义View的时候在事件传递这一块会遇到很多奇怪的问题，虽然每次google一下就基本找到了解决办法，却不能理解其事件传递的机制到底是怎么样的，看了无数的博客和文章，现在自己总结一下。

### 事件处理的方式
+ dispatchTouchEvent()------>传递事件
+ onInterceptTouchEvent()------>拦截事件
+ onTouchListener.onTouch()和onTouchEvent()------>消费事件

### 



