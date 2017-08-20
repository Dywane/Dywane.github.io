---
layout: post
title: "iOS开发 简化ViewController"
date: 2017-03-25
excerpt: "view controller通常是一个项目中最庞大的文件，因为它里面经常包含了不属于它的代码，同时这也使它成为代码中最难以重用的部分。所以为view controller瘦身，让其中的代码复用性更强，把相关代码放到正确的地方显得尤其重要。"
tag:
- iOS开发
- 翻译
comments: true
---

>翻译、修改自objc.io
>
原文链接：[Lighter View Controller](https://www.objc.io/issues/1-view-controllers/lighter-view-controllers/)

### 将Data Source和其他协议分离
为view controller瘦身最有效的方法就是把`UITableViewDataSource`中的代码移动到相关的类中，具体的方法可以参阅[《iOS应用开发 简明TableView》](http://www.jianshu.com/p/79619c56d9df)中的相关实现。

而更进一步，不只是TableView，这个方法可以扩展到其他的协议上，比如`UICollectionViewDataSource`。如果在开发中选择使用`UICollectionView`代替`UITableView`时，这个方法可以让你几乎不用修改viewController中的任何东西，甚至可以让Data Source同时支持两个协议，给予了极大的便利性。

### 将弱业务逻辑移到Model中
首先是代码，以下的代码是帮助用户查找优先事项的列表：

{% highlight objective_c %}
-(void)loadPriorities
{
	NSDate *now = [NSDate date];
	NSString *formatString = @"startDate <= %@ AND endDate >= %@";
	NSPredicate *predicate = [NSPredicate predicateWithFormat:formatString, now, now];
	NSSet *priorities = [self.user.priorities filteredSetUsingPredicate:predicate];
	self.priorities = [priorities allObjects];
}
{% endhighlight %}
然而，如果把这些代码移动到`User`类中会让它变得更加明晰，这时`ViewController.m`中会是：

{% highlight objective_c %}
-(void)loadPriorities
{
	self.priorities = [self.user currentPriorities];
}
{% endhighlight %}
而`User + Extensions.m`中则是：

{% highlight objective_c %}
-(NSArray *)currentPriorities
{
	NSDate *now = [NSDate date];
	NSString *formatString = @"startDate <= %@ AND endDate >= %@";
	NSPredicate *predicate = [NSPredicate predicateWithFormat:formatString, now, now];
	return [[self.priorities filteredSetUsingPredicate:predicate] allObjects];
}
{% endhighlight %}

将这些代码移动的根本原因是因为`ViewController.m`是大部分业务逻辑的载体，本身代码的复杂度已经很高，所以这类跟业务关联不大的代码比如日期转换、图像裁剪、设定过滤器等的操作可以分离到各自的类中完成，一方面为viewController减负，另一方面也能增进代码的复用。
>关于这个标题的翻译我斟酌了比较久的时间，因为在原文中是**“Move Domain Logic into the Model”**，意为“把**领域逻辑**移到Model中”。对于**“领域逻辑”**一词我进行过考究，大致意思为**“稳定的、不会改变的逻辑关系”**，同时在原文中也是使用了`NSPredicate`作为例子引用，而我认为其例子中的代码也是与业务相关的，只不过关联性不大，而且不会轻易改动，所以使用了**“弱业务逻辑”**一词代替了**“领域逻辑”**一词。

### 把数据处理的逻辑移到服务层
一些代码可能没办法很有效的移动到model中，然而这些代码却和model中的代码有清晰的关联，对于这种问题，可以使用`Store`。比如在下面的代码中，viewController需要完成从一个文件中获取一些数据，并对其进行操作：

{% highlight objective_c %}
-(void)readArchive 
{
	NSBundle *bundle = [NSBundle bundleForClass:[self class]];
	NSURL *archiveURL = [bundle URLForResource:@"photodata" withExtension:@"bin"];
	NSDate *data = [NSData dataWithContentsOfURL:archiveURL options:0 error:NULL];
	NSKeyedUnarchiver *unarchiver = [[NSKeyedUnarchiver alloc] initForReadingWithData:data];
    _users = [unarchiver decodeObjectOfClass:[NSArray class] forKey:@"users"];
    _photos = [unarchiver decodeObjectOfClass:[NSArray class] forKey:@"photos"];
    [unarchiver finishDecoding];
}
{% endhighlight %}
事实上，view controller不需要清楚怎么实现这些东西，而应该将这些处理交给一个`store object`来完成。

通过对代码进行分离，能够增进代码复用、对代码进行单元测试、保持view controller整洁等。同时能够让view controller更多关注于业务本身的内容，把数据的读取 、缓存、新建等操作交给服务层来处理。

### 把网络服务的逻辑移到Model层
这与上面提到的十分相似：**不要把网络服务的逻辑放到view controller中**，而应该把它们存放到不同的类中。

对于view controller，应该只是使用一个`completion block`来调用这些方法，而把网络请求、错误处理、缓存处理交给这些类来完成

### 把处理view的代码移到view层
无论是使用`Storyboard`还是纯代码编写view，创建复杂的view的任务不应该交给view controller。

比如在需要创建一个日期选择器时，更好的方法是把这些代码放到`DatePickerView`中，而不是放到`ViewController`中，和之前一样，是为了复用和简便。至于如何使用`storyboard`对view进行设置这里不再赘述。

### Communication（通讯）
view controller中最常发生的就是**通讯**，包括了和view层的通讯、和model层的通讯、和其他view controller的通讯等。尽管这是一个view controller必须做的事情，这里依然有办法能够对代码进行缩减。

对于view controller和view、view controller和model之间的通讯已经存在大量的优秀经验，比如使用KVO键值模式传值，或者使用`Core Data`中的`NSFetchedResultsController`等。然而，对于view controller之间的通讯的相关方法却比较少。

比如在我现在做的项目中，一个view controller需要根据使用者身份的不同（家长／老师）来对view controller的 state 进行不同的设置，而这个 view controller 又在不同的 state 下与不同的 view controller 通讯，传递不同的值。在这种情况下 view controller 中的代码会变得相当臃肿和混乱，所以正确的做法是把这些不同的 state 放到不同的 object 里面，再把它们传给 view controller ，让它们根据这个 state 来进行设置和修改，使我们不再需要被累赘的委托方法搞得十分混乱。

而在另一种情况下，各个 view controller 之间的跳转逻辑十分复杂，存在着严重的横向依赖，在这种情况下就不适宜使用普通的页面跳转模式，而应该使用**“中介者模式”**，创建一个`coordinator controller`，让它来管理页面跳转的逻辑。

---

## 结论
事实上现在有各种各样的方法为 view controller 减负，它们无一例外都是向着一个目标前进：写出可维护的代码，只有把这些方法灵活运用，熟记于心，才能真正避免弄出笨重而又难以维护的 view controller。
