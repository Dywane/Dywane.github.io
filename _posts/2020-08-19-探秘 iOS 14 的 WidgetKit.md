---
layout: post
title: "探秘 iOS 14 的 WidgetKit"
date: 2020-08-19
excerpt: "Widget Extension 作为iOS 14新加入的控件，提供了一种新的桌面展示方式，在主屏幕中及时展示用户关心的数据。"
tag:
- iOS开发
comments: true
---

## What’s new in Widget

- 能主屏显示
- 多种尺寸
- 不可交互，只可点击
- 智能叠放

Widget Extension 提供了 small, medium, large 三个尺寸，不同尺寸可以展示不同的数据、不同的界面，开发者也可以锁定自己APP的 Widget 只有某类尺寸，相同的widget也能重复添加。作为添加在主屏幕上的控件，苹果用了 **“At a glance”** 来形容 widget ，所以 widget extension 是无法交互的，它能做的只有展示一些信息与点击两个作用，点击后就会引导至app，同时为了性能与耗电量的考虑，Widget extension 也不能展示视频和动态图像。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb0dce91f85a46b4be2f7b1cc013efaa~tplv-k3u1fbpfcp-zoom-1.image)

另外用户可以将数个 Widget 放在同一个位置，这样 Widget 就会叠放在一起，它被叫做`Widget Gallery`，用户可以自由的切换顶部展示的 Widget。同时用户可以开启智能叠放，之后苹果会根据用户习惯等，在叠放的地方根据不同情况展示不同 Widget，如在每天早晨会展示天气 Widget，随后上下班通勤时会展示路况信息的Widget。开发者是无法指定自己的 Widget 在智能叠放中的位置的。

## What's new in Widget Development

以上的特定更多是产品上的表现形式，作为开发者我们更关心的是新的 Widget Extension 需要怎么开发：

- Swift UI Only
- Timeline Update
- IntentConfiguration
- Link

首先 Widget extension 只能通过`Swift ui`来开发，是没有办法调用任何`UIKit`的元素。只能使用`Swift UI`是一个大胆但也很合理的尝试，因为它很方便的**适配暗黑模式、动态字体**等元素，用在Widget开发上能够确保都能符合苹果的设计理念，同时iOS、iPadOS、macOS上的 Widget 也能利用同一套代码。

另外 widget extension 的刷新是基于**时间线**（Timeline）的刷新，不再是原有 Widget 基于`UIViewController`的各种刷新时机了，比如一个股票的widget，在交易的时候可以频繁更新，但是在非交易时段是可以完全不更新的，所以根据这个规律，开发者可以提前设置这个widget 的更新时间线，然后系统就会在对应的时间节点去自动更新 widget。

而 Widget 的展示的内容也是**可以让用户去定制**的，比如天气 widget 可以让用户去选择展示的城市，这个定制的选项是使用了 `Intents` 这个框架，这个框架最开始是在`SiriKit`上使用的，开发者是不需要编写代码，只需要加相关的配置项就可以自动生成界面和选项了。

最后 Widget 点击的跳转是通过一个叫`Link`的类去处理，类似`NSUR`L。

## Timeline Reload

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2859c0b67f65476f868324f7e0426911~tplv-k3u1fbpfcp-zoom-1.image)

在新的 Widget 中，刷新是基于 Timeline 的，Timeline是一系列`Timeline entry`的集合，`Timeline entry`包括了时间节点与对应时间节点 Widget 要展示的信息，Widget 的刷新完全由 `WidgetCenter`控制。开发者无法通过任何 API 去主动刷新 Widget 的页面，只能告知 `WidgetCenter`，Timeline 需要刷新了。

系统提供了两种方式来驱动 Timeline 的刷新。`System Reloads（系统刷新）` 和 `App-Driven Reloads（App通知的刷新）`

###System Reloads

- 可预测事件
- 按时发起
- 动态决策

System Reloads 通常是用于可预测事件的刷新，如股票 Widget ，他能预知每个交易日的开始时间和结束时间，所以开发者可以提前告诉`WidgetCenter`，它需要在什么时候刷新该 Widget 的时间线，在什么时候可以停止刷新。

这个行为由系统主动发起，会刷新 Widget 的 Timeline ，向 Widget 请求下一阶段刷新的数据。系统会按时发起 System Reloads ，比如天气 Widget 设置了每天刷新一次，那么在对应的时间节点就会刷新；另外系统还会动态决策每个不同 Widget 的 TimeLine 的系统刷新频次，如果某个 Widget 经常被查看，那么它被刷新的频率将高于查看次数相对较低的 Widget 。

### App-Driven Reloads

- 主动刷新
- 主APP必须存活

指的是 App 主动请求 Widget 下一阶段刷新的数据。这里也要分两种场景，应用在前台运行和应用在后台运行。当应用在前台运行的时候，App 可以直接请求`WidgetCenter`的 API 来触发 Reload Timeline；而当应用处于后台时，后台推送（Background Notification）也可以触发 Reload Timeline。

### Timeline Provider

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66f8c9d58df348268f11f9c10df35337~tplv-k3u1fbpfcp-zoom-1.image)

既然上文说到这个 **“Reload”** 并不是说刷新 Widget 的界面，而是说请求新的 Timeline 数据，那么`TimelineProvider`就是提供这个数据的对象。`TimelineProvider`提供的数据有两部分，一部分是`TimelineEntry`，另外一部分是 `ReloadPolicy`。

- TimelineEntry：包括了时间点与对应时间节点下 Widget 需要呈现的信息。
- ReloadPolicy：接下来这段时间 Timeline 的刷新策略，一共有三种：
 - atEnd: 是指 Timeline 执行到最后一个`TimelineEntry`的时候再刷新。
 - atAfter: 是指在某个时间以后有规律的刷新。
 - never：是指以后不需要刷新了，什么时候需要重新刷新需要 App 重新告知 Widget。

当`TimelineProvider`提供完下一阶段的数据之后，就会停止运行。系统会根据`TimelineEntry `的信息，在对应的时间节点对 Widget 的展示内容进行刷新。由于开发者只提供了`TimeLineEntry`和`ReloadPolicy`，所以 Widget 界面真正的刷新时间和刷新频率其实全部都是交给系统控制的，这样系统能够更好的把控 Widget 的耗电量与刷新频率，避免某个 Widget 由于大量的刷新导致消耗了过多的电量。

## Intent Configuration

- 无需编写UI代码
- 用户设置后会刷新 Widget 界面

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee96dbbde3b64084bb58533b55b4fda7~tplv-k3u1fbpfcp-zoom-1.image)

在用户使用的过程中，Widget 是需要一定的自定义能力，比如当我添加一个天气 Widget，我可以选择我 Widget 要显示哪个地方的天气。为了实现这个能力，苹果给 Widget 提供了 Configuration 的能力。顾名思义，就是可配置。一共有两种配置类型：

- StaticConfiguration：静态配置，只和用户信息有关系
- IntentConfiguration：动态配置，支持用户自定义

`IntentConfiguration`的实现是基于 `Intents.framework`，开发过 `SiriKit` 和 `Shortcuts` 的同学一定知道 Intents API 是用于了解用户意图的。简要的说它是一个智能的表单系统，创建一个 SiriKit Intent Definition File 之后，只需要简单的配置，Xcode 会自动帮你生成对应的代码和UI。

## Link

Widget 的 UI 是不支持滚动等交互元素的。唯一开放的能力只有通过点击或DeepLink 来唤起主 App。

苹果提供了两种 API 给到开发者，第一种是SwiftUI 的 `WidgetURL API`，它的可点击区域是在整个widget页面。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4bcab6bc599a4b8eaa7307b3da4e6ed2~tplv-k3u1fbpfcp-zoom-1.image)

对于 systemSmall 类型来说，只支持 widgetURL 的方式，但是对于 systemMedium 和 systemLarge 还可以使用 SwiftUI Link API，而 Link 的可点击区域是这样的：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/137de66b32814372bb22c533fab17480~tplv-k3u1fbpfcp-zoom-1.image)

## 总结

新的 Widget 带来了更多的信息展示窗口，APP能够通过 Widget 在主屏幕上更好的展示数据，其新特性也是十分吸引与有趣。另外需要注意的是新 Widget 目前还是在Beta版本，接口尚不稳定（如原有的`PlaceHolder`就被废弃了），开发时还需要注意这方面的坑。

## 参考

- [Apple Widget：下一个顶级流量入口？](https://mp.weixin.qq.com/s/ujZfU1CEQ1EfqoO8UR_kSg)
- [Widgets - System Capabilities](https://developer.apple.com/design/human-interface-guidelines/ios/system-capabilities/widgets)
- [Widgets code-along](https://developer.apple.com/news/?id=yv6so7ie)
- [Creating a Widget Extension](https://developer.apple.com/documentation/widgetkit/creating-a-widget-extension)
