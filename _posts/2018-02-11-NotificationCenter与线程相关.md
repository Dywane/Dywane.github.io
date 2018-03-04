---
layout: post
title: "NotificationCenter与线程相关"
date: 2018-2-11
excerpt: "NotificationCenter是基于iOS程序内部之间的一种消息广播机制，主要是为了解决应用程序不同对象之间通信解藕而设计。"
tag:
- iOS开发
comments: true
---

> `NotificationCenter`是基于iOS程序内部之间的一种消息广播机制，主要是为了解决应用程序不同对象之间通信解藕而设计。它基于`KVO`模式设计，当收到通知后由通知中心根据转发表将消息发给观察者。  

在Apple的文档里面有如下说明：

>A notification center delivers notifications to observers synchronously. In other words, when posting a notification, control does not return to the poster until all observers have received and processed the notification. To send notifications asynchronously use a notification queue, which is described in Notification Queues.
>
>In a multithreaded application, notifications are always delivered in the thread in which the notification was posted, which may not be the same thread in which an observer registered itself.

其实它主要是指`NotificationCenter`是一种**“同步的”**消息机制，当一个通知被post之后，他将会在所有的观察者都完成了对应的方法后才会返回。在多线程的应用中，通知是在哪个线程Post就在哪个线程分发，也就是在同一个线程被观察者处理，所以观察者的回调函数是完全由发送消息的线程决定，而不是由注册时所在的线程决定。 

关于这一点我举一下我在对聊天消息进行优化的时候发现的问题作为例子，在原来的代码结构中，需要处理获取的消息，并更新会话cell的排序：

### swift

#### ConversationModel.swift
```swift
func handleMessage() {
	var message = Message()
	/* do something to create and process messages */
	NotificationCenter.default.post(name: NSNotification.Name(rawValue: kRefreshConversation), object: self)
}

```
#### ConversationVC.swift
```swift
func viewDidLoad() {
	NotificationCenter.default.addObserver(self, selector: #selector(refreshConversatin(_:)), name: NSNotification.Name(rawValue: kRefreshConversation), object: nil)
	/* other things ...*/
}

@objc private func refreshConversatin(_ noti: Notification) {
	/* use a long time to sort conversation */
	tableview.reloadData() 
}

```

在原来这种处理方式中，由于`refreshConversation`这个方法会耗费大量时间，而这个通知是在主线程发起的，所以会导致主线程被阻塞，所以我针对这个问题对`ConversationModel`进行了重构。

首先我将Conversation获取并组装Message的这一块内容都放置在后台线程进行，因为Model相关的操作可能会占用很多的时间，放在主线程创建并组装是不可靠的。当message组装完毕需要更新`ConversationVC`中的排序时，在`ConversationVC`中接收到通知后再转到主线程进行操作。同时我在排序的代码中也进行了优化，如果`ConversationVC`没有被展示的话，就不需要占用主线程调用`tableView`的`tableView.insertRows`方法，仅仅需要在后台线程中对`ConversationModel`的数组进行排序即可。因为`tableView.insertRows`会占用大量主线程时间，所以减少使用该方法的频率，只在必要时候调用，将排序等操作放到后台线程进行是比较好的解决方法。 
