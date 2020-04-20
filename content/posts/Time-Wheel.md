---
title: "Time Wheel"
date: 2019-12-08T09:42:55+08:00
draft: true
---

## 参照 Kafka 实现一个 Go 的时间轮

> 最近理解了一下游戏业务中的延时器，同时对以前 Kafka 的定时器的设计心心念念，所以对比了一下业务中的优劣，决定实现一个 Kafka Go 版本的定时器。

#### 定时器

`概念：` 我们的业务中，或多或少会出现定时任务，比如，活动发布后三小时结束；对于这类定时任务，每种语言都有其自己的实现，比如 js setTimeOut()/setInterval()， java Timer/TimerTask 包，Go Timer 包等。每种语言对于定时器的实现都根据自身语言的特性有所差异。

`时间轮：` 一种实现定时器的数据结构。[参考]([https://crossoverjie.top/2019/09/27/algorithm/time%20wheel/](https://crossoverjie.top/2019/09/27/algorithm/time wheel/))

`Kafka 实现：` delayQueue + 层级时间轮。

`当前业务实现:`  层级时间轮。

#### 对比分析

> 为什么要参照 Kafka 来实现呢，那就是对比分析啦。

