---
title: requestAnimationFrame和requestIdleCallback小记
date: 2019-12-13 11:34:05
tags:
---

在学习react fibber中了解到fibber实现主要就是利用requestAnimation和requestIdleCallback这两个api，本文主要就是简单的总结下这两个api。

## 一. window.requestAnimationFrame
说到实现动画效果，requestAnimationFrame一定是出现在脑海里的第一个词。相比于setTimeOut来实现动画时间间隔的不可控制，丢帧现象的普遍。requestAnimationFrame要求浏览器在下次重绘之前调用指定的回调函数更新动画。也就是说他的回调执行的频率是和系统的帧率保持一致的（大约60Hz左右）。它就像是浏览器专门为我们实现动画效果开的绿色通道。

## 二. window.requestIdleCallback
我们都知道页面是一帧一帧渲染出来的，每秒60帧刚好是一个人视觉上流畅的帧率，小于这个值时，用户就会感觉到页面卡顿，也就是每一帧分到的时间是（1/60）s,约16ms。当每一帧的任务执行时间超过16ms就会出现卡顿现象。

浏览器每一帧需要做的工作大概是

- 处理用户交互
- js解析执行
- 帧开始。窗口尺寸变更，页面滚动处理等
- requestAnimationFrame
- Layout
- Paint
在两个执行帧之间，浏览器通常有一小段空闲时间，requestIdleCallback可以在这个空闲期（Idle Period）调用空闲期回调（Idle Callback），执行一些任务。

