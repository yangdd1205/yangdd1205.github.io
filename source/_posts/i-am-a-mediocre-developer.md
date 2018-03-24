---
title: 【译】我是一个平庸的开发者
tags:
  - 人生经验
categories: 译文
date: 2018-03-24 21:49:24
---

> 原文作者：Nikita Sobolev
地址：https://dev.to/sobolevn/i-am-a-mediocre-developer--30hn

我个人认识一些非常有天赋的开发人员，他们可以轻而易举的创造出精彩的软件。因为有着这些有天赋人的存在，我们的行业充满了很高的期望。但可悲的事实是：不是每个人都是忍者/大师/摇滚明星开发者。

而我有自知之明：我是一个平庸的开发者。 如果你不是天才，那么本文将指导你在行业中生存下去。

<!-- more -->

## 我总是谷歌简单的事情

我记不清很多东西。像标准库中的函数和方法，参数位置，包名称，样板代码等等。

所以，我不得不谷歌。这就是我的日常。我也重复使用旧项目的代码。有时我甚至从 StackOverflow 或 Github 复制粘贴。这是一件真实的事情：[StackOverflow 驱动开发](https://meta.stackoverflow.com/questions/361904/what-is-stack-overflow-driven-development)

并不是我一个人这样，许多其他开发人员也这样做。有一个由 `Ruby on Rails` 创造人在 twitter  上发起的[讨论](https://twitter.com/dhh/status/834146806594433025)。

为什么这一点是不好的呢？它有几个缺点：
1. 复制的代码可是是糟糕的设计或者易受攻击的代码，没有质量保证
2. 形成了一个特定的心态：如果我们不能谷歌东西，那么“休斯顿，我们有一个问题”
3. 断网时，我们将无法工作

但是，我认为这不是一个大问题。它甚至可以作为你的秘密武器。我有一些建议要遵循，以减少负面影响。

### 生存之道

1. 使用 `IDE` 来获得自动补全和建议，所以你不必谷歌语言基础知识
2. 记住你已经解决了这个问题的地方（而不是如何解决）。所以你可以随时在那里寻找解决方案
3. 粘贴到项目中的所有代码都应该进行分析、重构，并在以后进行评审。这样我们就不会用错误的代码来伤害项目，而是用快速解决方案来帮助它

## 我保持简单

机器总是按照它们被告知的去做，有时候它们只是被告知去做错误的事情。所以，软件开发中的主要问题不是机器，而是开发人员的思维能力。这是非常有限的。因此，我们平庸的开发人员不能浪费它来创建复杂的抽象的 、晦涩的算法或不可读的长代码块。[保持简单就好](https://en.wikipedia.org/wiki/KISS_principle)。

但是，我们怎么来区分代码的简单或者复杂？我们需要使用 `WTFs/Minute` 来测试代码质量。

![WTF](https://res.cloudinary.com/practicaldev/image/fetch/s--RTnc6j1i--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://i2.wp.com/commadot.com/wp-content/uploads/2009/02/wtf.png%3Fresize%3D550%252C433)

这个原则很容易理解。每当你在代码中发现一些你不明白的东西时——这太复杂了。你能做什么？
1. 改写，让它更简洁
2. 提供文档
3. 加注释到最棘手的部分。但是请记住，注释是代码的味道

### 如何开始写简单的东西

1. 使用正确的变量名、函数名和类名
2. 确保程序的每个部分[只做一件事](https://en.wikipedia.org/wiki/Single_responsibility_principle)
3. [纯函数](https://en.wikipedia.org/wiki/Pure_function)优于非纯函数
4. 非纯函数优于类
5. 只有在非常有必要的情况下才返回类


## 我不相信自己

一些开发人员证明可以提供高质量的代码。比如她：阿波罗计划的首席软件工程师 Margaret Hamilton。图片中，她站在她为月球任务编写的代码旁边：
![Margaret Hamilton](https://res.cloudinary.com/practicaldev/image/fetch/s--G4sZPf_1--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/http://cdn8.openculture.com/2017/08/29205628/margaret-hamilton-mit-apollo-code_0.jpg)


但是，每当我编写任何代码时，我都不相信自己。即使在项目最简单的部分，我也可以把事情搞得很糟糕。可能包括：
1. 语法错误
2. 逻辑错误
3. 设计错误
4. 样式错误
5. 安全错误
6. WTF 错误（我的最爱！）

并没有关于“学习如何编写没有 bug 的代码”的魔法书，这是正常的。[所有软件都有 bug](https://m.signalvnoise.com/software-has-bugs-this-is-normal-f64761a262ca)。除了这个[框架](https://github.com/kelseyhightower/nocode)。

问题是：任何人都不应该被允许编写有明显错误的代码。至少，我们应该尝试。但是我怎么才能保护我自己的项目？有多种方式可以做到这一点。

### 生存之道

1. 编写测试。写很多测试。从集成测试开始到单元测试。在每次拉取请求前在 `CI` 中运行它。这将保护你免受一些逻辑错误
2. 使用静态类型或可选的静态类型，举个例子，我们 使用 [mypy](http://mypy-lang.org/)，[flow](https://flow.org/)。积极的影响：简洁的设计和编译时检查
3. 使用自动样式检查。每种语言都有很多风格检查器
4. 使用质量检查。有些工具在你的代码库上运行一些复杂的启发式算法来检测不同的问题，比如这行内部有太多的逻辑，这个类是不需要的，这个函数太复杂了
5. 评审的代码。在合并到 `master` 之前，和合并后的某个时间
6. [支付报酬请其他人对您的代码进行评审](https://wemake.services/meta/rsdp/audits/)。这项技术具有巨大的积极影响！因为当开发人员首次查看您的代码时，他们更容易发现不一致和糟糕的设计决策

## 它不应该只能在我电脑上运行

![](https://res.cloudinary.com/practicaldev/image/fetch/s--h027pWFE--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://www.ca.com/us/products/excuse-free-testing/worked-fine-on-my-machine/_jcr_content/page/adaptiveimage_855e.img.620.high.jpg/1484844865861.jpg)

大约十年前，当我的团队开发了我们的第一个大型软件项目时，我们已经将它作为 `java` 源文件发布。它无法在目标服务器上编译。这是提交给客户之前的几个小时。这是一个很大的失败！不知怎的，我们已经设法启动并运行，但这是一次改变人生的体验。

发生这种情况是因为构建流水线中有很多配置和复杂的事情。而且我们无法妥善管理这个系统的复杂性。从那一天起，为了减少这一步的复杂性，我尝试在孤立的环境中打包我的程序。并且在实际部署发生之前在这个环境中测试它们。

在 `docker`（和一般容器）兴起的过去几年里，它变得如此简单。`docker` 允许您在相同的隔离环境中运行开发，测试和生产。所以，你永远不会错过任何重要的事情。

伤了你？谈论我自己时，我总是在创建服务器，初始配置它们或将它们连接在一起时忘记了一些事情。有很多事情要记住！希望我们仍然可以实现自动化。有不同的很棒的工具来自动化您的部署过程。如：[terraform](https://www.terraform.io/)、[ansible](https://www.ansible.com/) 和 [packer](https://www.packer.io/)。阅读关于他们的信息，找出您实际需要哪一项任务。

我也尝试尽快建立 [CI](https://about.gitlab.com/features/gitlab-ci-cd/) / [CD](https://about.gitlab.com/features/gitlab-ci-cd/)。因此，如果我的构建在测试或部署中失败，我能收到报告。

### 生存之道

1. 自动化您用于部署的任何内容
2. 使用 `docker` 进行应用程序的开发、测试和部署
3. 使用部署工具

## 应用程序部署后，我仍然不相信自己

哦，最后，我的应用程序正在生产中。它正在工作。我可以小睡一下，什么都不会被打破。等等，不！一切都将打破。是的，我的意思是：一切。

实际上，有一些工具可以更容易地查找和解决现有问题。

1. [Sentry](https://sentry.io/welcome/)。当您的任何用户发生错误时 - 您将收到通知。已经绑定了几乎所有的编程语言
2. 将多个进程和服务器的日志收集到一个地方的不同[服务](https://papertrailapp.com/)和[工具](https://www.elastic.co/products/kibana)
3. [服务器监控](https://grafana.com/)。可以配置 CPU、磁盘、网络和内存的监控信息。您甚至可以在用户真正中断您的服务之前确定可伸缩的时间

**简单来说**，我们需要监控在生产中的应用程序。我们有时使用所有这些工具，有时只使用最需要的部分。

## 不断学习

哇，有太多的东西需要学习。但这就是它的工作原理。如果我们想编写好的软件，我们需要不断学习如何去做。没有简短的方法或魔术技巧。只要学会如何每一天都变得更好。

总之，我们需要理解两件基本的事情：
1. 每个人都会遇到问题。唯一重要的是我们为这些问题做好了准备
2. 我们可以将问题的根源降低到一些可接受的水平

这与你的心智能力或心态无关。
