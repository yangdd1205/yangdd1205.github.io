---
title: 【译】模块化应用架构——管道
tags:
---
> 原文地址：https://www.goetas.com/blog/modular-application-architecture-pipelines/

这是如何构建模块化和可扩展应用程序系列的第三篇文章。我们将开始着眼于如何使用“管道”实现插件系统。有些实现称它们为“中间件”。

<!-- more -->

与事件相比，管道可以在不同的情况中使用。

事件频繁地使用在需要添加功能的情况下（只有当事件类型支持它时，才能更改现有功）。而管道一般用于添加数据或者修改现有数据。

一般来说，对于“管道”，我们只是将一些数据通过一系列转换。

![pipeline](https://www.goetas.com/img/posts/plugin-based-architecture/pipeline.png)

有些实现允许您决定执行的顺序，有些则不允许，有些允许您在运行时更改顺序。

由于执行顺序可能很重要，所以“插件注册”步骤非常重要（有关详细信息，请参阅第一篇文章）。

## 案例


