---
title: 【译】模块化应用架构——事件
tags:
---
> 原文地址：https://www.goetas.com/blog/modular-application-architecture-events/

这是描述构建模块化和可扩展应用程序的策略系列的第二篇文章。在这篇文章中，我们将开始研究如何使用“事件”来实现插件系统。

<!-- more -->

实现基于插件的体系结构最常见方法之一是使用“事件“。其思想是应用程序核心”抛出“事件，模块监听这些事件并做出相应的反应。

模块（通常称为插件）也可以”抛出“事件，使整个机制更强大，因为可以在不直接由核心协调的模块之间构建某种通信。

事件监听器可以对事件作出反应，并最终将一些数据返回到核心。核心应用程序可以使用接收到的数据来实现某个目标（例如向用户显示它们或使用它们”抛出“其他事件）。

这种策略被流行的框架作为 [Symfony](https://symfony.com/) 或 [WordPress](https://wordpress.com/)，WordPress 将这种方法称为”钩子“，而 Symfony 和 Laravel 将它们称为”事件“，它们由”事件调度器“管理。

![events](https://www.goetas.com/img/posts/plugin-based-architecture/events.png)


## 事件调度器

在这里，我们将详细分析核心应用程序如何抛出事件、插件如何监听它们以及如何使用该机制构建插件系统。

1. 应用程序核心抛出事件。为了使系统灵活高效，通过名字来区分事件，因此事件监听器可以在特定事件上注册，而不必监听由核心抛出的每一个事件。
2. 事件可能有一个有效负载。有效负载包含与事件相关的信息。这样，插件就会有更多关于事件被抛出的上下文的信息，而仅仅是名字是不够的。
3. 事件监听可以将一些数据返回给已抛出事件的用户。返回的数据应该是一种可以由谁抛出事件来理解的格式。


### 事件特性

可以在事件系统中添加或删除一些特性，这将给我们提供一些交换特性或限制。即使是“名称”、“有效载荷”和“返回”特性，也可以从事件系统中删除，使其极小化。

* 删除名称：删除事件名称，将使我们的系统更加复杂，性能更低，因为监听器不得不监听所有的事件，并决定是否对事件做出反应。删除事件名称是一个不常见的选择，因此不推荐。
* 删除有效负载：删除有效负载，将使我们的事件变得不那么有用，因为只有监听器了解事件上下文的唯一信息就只有名称。监听器不得不寻找其他方法来获取上下文信息。
* 删除返回：删除返回，将将使得事件监听器不能与核心应用程序交互。对系统的影响，必须由插件使用其他方法直接实现。核心应用程序不能直接“使用”插件的效果。返回值是常见的，如果没有返回值，则更容易引入异步事件。
* 添加停止事件传播：给予事件监听器停止事件传播的能力（通过返回一个特殊的值作为例子），将有可能影响由核心抛出的“事件”效应到其他插件上。
* 添加监听器优先级：添加监听器优先级，可以让插件更好地合作，特别是在“负载”和“停止事件传播”特性可用的情况下。可以决定事件侦听器的执行顺序，或者只执行一个事件侦听器，即使在同一事件上注册了多个事件。
* 添加通配符事件监听器：在需要听多个事件、所有事件或只知道部分名称的事件时，通配符事件监听是很有用的。一个常见的情况是分析插件或调试功能。
* 添加抛出事件的能力：让插件能够抛出事件是一个有用的特性，它允许加强“插件的插件”使整个系统更加灵活。太多的灵活性的缺点是很容易失控。

**异步事件**

如果我们决定只将“名称”和“有效负载”放入事件系统中，就可以很容易地将其转换为异步事件系统。这是因为每个事件都独立于其他事件（这对于具有同步流的 Web 系统应用程序来说是有效的，而对于桌面程序或多线程/并发语言来说，情况则完全不同）。

### 事件监听注册器

正如第一篇文章中所说，一个基本步骤就是插件注册。可以使用发现和配置策略。

## 示例

实现事件系统是一个相当常见的任务，这意味着已经有许多实现我们可以使用，而不是重新发明轮子。

他们都有特定的功能，只是选择最适合我们。如下所示，所有这些功能都可以为您提供稍微不同的功能，并且行为略有不同，但通常它们非常相似。

我们将看到如何实现具有由插件动态创建的内容的边栏。我们将看到如何使用Symfony，Laravel和Wordpress来做到这一点。

**Symfony**

```PHP
<?php
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\EventDispatcher\Event;

// the event and payload
class SidebarEvent extends Event
{
    private $placement;
    private $items = [];
    public function __construct($placement)
    {
        $this->placement = $placement;
    }

    public function getPlacement()
    {
        return $this->placement;
    }

    public function addItem(Item $item)
    {
         $this->items[] = $item;
    }

    /**
     * @return Item[] 
     */
    public function getItems() : array
    {
        return $this->items;
    }
}


// listeners/plugins
$addContactListener = function (SidebarEvent $ev) {
    $ev->addItem(new Item('Contact on the ' . $ev->getPlacement()));
};
$addHomeListener = function (SidebarEvent $ev, $eventName, EventDispatcher $dispatcher) {
    $ev->addItem(new Item('Homepage' . $ev->getPlacement()));
};

// here is the application
$dispatcher = new EventDispatcher();

// listener registration
$dispatcher->addListener('site_bar', $addContactListener, -100); // priority
$dispatcher->addListener('site_bar', $addHomeListener);

// application throwing events
$dispatcher->dispatch('site_bar', $event = new SidebarEvent('left'));
$items = $event->getItems();

// example use the return values
echo "<li>";
foreach ($items as $item) {
    echo "<ul>" . $item->getTitle() . "</li>";    
}
echo "</li>";
```

* `$dispatcher` 变量是标准的 symfony 事件调度程序；symfony 事件调度程序是同步的，它允许监听程序停止事件传播或分派/抛出新事件。
* 我们使用一个定制的 `SidebarEvent` 事件，它具有 `$placement`（表示我们在哪里是侧边栏）作为有效载荷并允许数据作为返回值;设置返回值将停止事件传播。
* 我们的事件监听器是 `$addContactListener` 和 `$addHomeListener` 作为简单的回调函数。
* 事件监听器使用 `addListener` 方法在 `site_bar` 和 `admin_site_bar` 事件中注册。
* 我们的应用程序抛出一些事件（使用调度方法）；在这种情况下，我们使用特定的事件类（`SidebarEvent`），并且使用页面 url 作为负载。
* `$items` 将包含调用 `setItems` 方法时由事件侦听器最终设置的值；`$items` 将是类型提示强制执行的 `Item` 数组。

**Laravel**
```PHP
<?php

use App\Events\SidebarEvent;  

// the event and payload
class SidebarEvent
{
    private $placement;
    public function __construct($placement)
    {
        $this->placement = $placement;
    }

    public function getPlacement()
    {
        return $this->placement;
    }
}

// listeners/plugins
$addContactListener = function (array $payload) {
    list($ev) = $payload;

    return 'Contact on the ' . $ev->getPlacement();
};
$addHomeListener = function ($payload) {
    list($ev) = $payload;

    return 'Homepage on the ' . $ev->getPlacement();
};

// listener registration
Event::listen('site_bar', $addContactListener);
Event::listen('site_bar', $addHomeListener);

// application throwing events
$items = Event::fire('site_bar', [new SidebarEvent('left')]);

// example use the return values
echo "<li>";
foreach ($items as $item) {
    echo "<ul>" . $item . "</li>";    
}
echo "</li>";
```
* 该事件是默认的 laravel 外观；laravel 事件系统也是同步的，有一个有效载荷（默认情况下必须是一个数组），支持广播，通配符事件和返回值。
* 我们正在使用具有 `$placement` 作为有效负载的自定义 `SidebarEvent` 事件。
* 我们的事件监听器是 `$addContactListener` 和 `$addHomeListener` 作为简单的回调函数。返回 `false` 将停止事件传播。
* 事件监听器使用 `Event::listen` facade 方法在 `site_bar` 和 `admin_site_bar` 事件中注册。
* 我们的应用程序会抛出一些事件（使用 `Event::fire` facade 方法）；在这种情况下，我们使用特定的事件类（`SidebarEvent`），并且使用页面 url 作为负载。
* `$returnPageView` 将包含一个数组，其中包含所有事件的所有非错误返回值。由于 `$returnPageView` 中的值不能通过键入提示来检查，因此应该在应用程序级别进行健全性检查。


**WordPress**
```PHP
<?php

// listeners/plugins
$addContactListener = function ($placement) {
    echo "<ul>Contact on the $placement</li>";
};
$addHomeListener = function ($placement) {
    echo "<ul>Homepage on the $placement</li>";
};

// listener registration
add_action('site_bar', $addContactListener); 
add_action('site_bar', $addHomeListener);

// application throwing events

echo "<li>";
$returnPageView = do_action('site_bar', 'left');
echo "</li>";
```
* 我们正在使用 WordPress 的基本行动事件系统；支持优先级，有效负载和返回值。
* 我们的事件监听器是 `$addContactListener` 和 `$addHomeListener` 作为简单的回调函数。
* 事件监听器使用 `add_action` 函数在 `site_bar` 和 `admin_site_bar` 事件上注册。
* 我们的应用程序会抛出一些事件（使用 `do_action` facade方法）；在这种情况下，我们使用一个简单的字符串作为负载，我们有页面 url。
* `$returnPageView` 将包含来自上一个事件侦听器的返回值；由于无法检查 `$returnPageView` 中的值，因此应该在应用程序级别进行健全性检查。

## 最后

“事件”插件机制通常在应用程序希望能够控制插件能够做什么的时候使用。插件只能在应用程序决定的“扩展点”中与之交互。

优点：
* 非常灵活。
* 为应用程序提供的高度控制：应用程序决定扩展点和数据结构的使用。
* 可以是异步的。
* 透明：应用程序不需要知道注册了多少个插件以及它们的结构。
* 可文档化：可以显式定义返回和有效负载。

缺点：
* 仅限于预定义的扩展点：没有一种简单的方法可以与应用程序中不提供适当扩展点的部分进行交互。
* 透明：因为应用程序不知道插件的作用和它们的数量，所以很难保持应用程序的质量；“坏的”插件不能排除在外。
* 在某些调度器实现中，如果触发了侦听器，则很难理解。
* 很难调试，并且在错误发生时几乎无用的堆栈跟踪。

备注：根据您决定使用的实现，上述一些优点/缺点可能会改变。

事件是最流行的扩展机制之一，在下一篇文章中，我们将看到如何以及何时可以使用“中间件”扩展模式。