---
title: PDI 定时任务调度问题
date: 2024-01-13 21:53:50
updated: 2024-01-13 21:53:50
tags:
---

## 问题

假设有一个 PDI（Kettle）Job，运行这个 Job 需要耗时 30 秒钟。现在每 10 秒钟调度运行一次这个 Job。我们的问题是这个 Job 的运行情况是怎样的？具体地说是上一个调度的 Job 还没有执行完就到了下一个调度的 Job 的执行时间，那么下一个调度的 Job 会执行吗？也就是定时任务如果执行的时间很长的话，那么下一次还能按时执行嘛？

{% asset_img schedule.drawio.svg %}

* **Case 1**：无论上一次 Job 是否执行完成，下一次 Job 按照 10 秒钟的间隔准时开始；
* **Case 2**：上一次 Job 未执行完成，下一次 Job 不会开始执行；一旦上一次 Job 执行完成，下一次 Job 立即开始执行，中间没有间隔；
* **Case 3**：上一次 Job 未执行完成，下一次 Job 不会开始执行；上一次 Job 执行完成后等待 10 秒钟的间隔再开启下一次 Job 的执行；

Kettle 的定时任务是哪一种情况呢？

<!-- more -->

## 分析

我们先来问一下 ChatGPT，看看它怎么说？我们在三个不同的会话问 ChatGPT 同样的问题

第一个会话

{% asset_img chatgpt1.png %}

第二个会话

{% asset_img chatgpt2.png %}

第三个会话

{% asset_img chatgpt3.png %}

从这三个回答来看基本可以确定如果一个任务的执行时间很长，而下一次任务的触发时间点已经到了，调度器将会推迟触发下一次任务的执行。因此我们基本可以排除 Case 1 这种情况了，那到底是 Case 2 还是 Case 3 呢？我们需要做一点小实验。

## 实验

在这个实验中我们将创建只有两个节点的 Job，一个 Start 节点，这是 Job 必须的一个节点；一个 Wait for 节点，用来模拟耗时的任务。

{% asset_img job.png %}

我们给 Wait for 节点，即模拟任务节点，配置 30 秒钟的等待时间

{% asset_img wait-for.png %}

给 Start 节点配置没 10 秒钟调度一次，注意勾选“Repeat”复选框，不然只会调度一次

{% asset_img start.png %}

然后在本地运行起来

{% asset_img run.png %}

运行一段时间后把执行结果的日志复制出来进行分析，为了便于分析在日志中间加了一些空行来分割每一次调度

```text
2024/01/14 10:40:45 - Spoon - Spoon
2024/01/14 10:43:41 - Spoon - Spoon
2024/01/14 11:02:06 - Spoon - Starting job...
2024/01/14 11:02:06 - Job 1 - Start of job execution

2024/01/14 11:02:07 - Job 1 - Job 1
2024/01/14 11:02:17 - Job 1 - Starting entry [Wait for]
2024/01/14 11:02:47 - Job 1 - Finished job entry [Wait for] (result=[true])

2024/01/14 11:02:47 - Job 1 - Job 1
2024/01/14 11:02:57 - Job 1 - Starting entry [Wait for]
2024/01/14 11:03:27 - Job 1 - Finished job entry [Wait for] (result=[true])

2024/01/14 11:03:27 - Job 1 - Job 1
2024/01/14 11:03:37 - Job 1 - Starting entry [Wait for]
2024/01/14 11:04:07 - Job 1 - Finished job entry [Wait for] (result=[true])

.
.
.
```

在完成执行前的准备工作后，02:07 开始等待 10 秒钟直到 02:17 开始执行 Wait for 节点直到 02:47 耗时 30 秒钟后执行完成，这是第一次调度的时间节点。接着等待 10 秒钟直到 02:57 开始执行 Wait for 节点直到 03:27 耗时 30 秒钟后执行完成，这是第二次调度的时间节点。接着等待 10 秒钟直到 03:37 开始执行 Wait for 节点直到 04:07 耗时 30 秒钟后执行完成，这是第三次调度的时间节点。如果我们不停止 Job 它将按照这样节奏一直运行下去。

## 结论

从实验结果可以看到 PDI 的定时任务的调度符合 Case 3 这种情况。在触发第一次调度的任务还未执行完成前即使到了第二次调度触发的时间点也不会立即触发任务执行，调度器将等待第一次调度的任务执行完成，然后等待一个触发周期再出发第二次调度。

在这里我们分析的是 PDI 自带的定时任务调度功能，如果是使用 Windows 的任务计划执行程序来调度则需要设置任务计划“请勿启动新实例”，具体参考 [3.kettle-定时执行任务](https://www.cnblogs.com/zdyang/p/11759873.html)；如果是使用 Linux 的 Crontab 功能则可能需要参考 [crontab定时任务第一个周期未完成下一个周期执行就来了](https://www.cnblogs.com/lemon-le/p/8476604.html) 控制调度器不等待任务执行完成就开始下一次调度的问题。

## 参考

1. [kettle的job定时任务的一个小问题](https://cloud.tencent.com/developer/article/1442132)
2. [Launching job entries in parallel](https://pentaho-public.atlassian.net/wiki/spaces/EAI/pages/370049045/Launching+job+entries+in+parallel)，提到了 Job Execution Engine 采用了 Backtracking Algorithm
3. [Kettle基本概念 之 Kettle设计模块](https://www.jianshu.com/p/a02480e2dda1)，多路径和回溯
4. [定时任务时间过长会不会影响下次的执行？](https://zhuanlan.zhihu.com/p/344053917)
5. [Springboot中上一个定时任务没执行完，是否会影响下一个定时任务执行分析及结论](https://blog.csdn.net/wtwcsdn123/article/details/124684997)
