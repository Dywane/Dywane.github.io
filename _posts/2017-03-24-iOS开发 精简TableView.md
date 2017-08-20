---
layout: post
title: "iOS开发 精简TableView"
date: 2017-03-24
excerpt: "TableView 是iOS app 中最常用的控件，许多代码直接或者间接的关联到table view任务中，包括提供数据、更新tableView、控制tableView行为等等。下面会提供保持tableView代码整洁和结构清晰的方法。"
tag:
- iOS开发
- 翻译
comments: true
---

>翻译、修改自objc.io
>
原文链接：[Clean Table View Code](https://www.objc.io/issues/1-view-controllers/table-views/)


## UITableViewController vs. UIViewController

### TableViewController的特性
table view controllers可以读取table view的数据、设置tabvleView的编辑模式、反应键盘通知等等。同时Table view controller能够通过使用`UIRefreshControl`来支持“下拉刷新”。

###Child View Controllers
tableViewController也可以作为child view controller添加到其他的viewController中，然后tableViewController会继续管理tableView，而parentViewController能管理其他我们关心的东西。


{% highlight objective_c %}
    -(void)addDetailTableView
    {
	    DetailViewController *detail = [DetailViewController new];
	    [detail setup];
	    detail.delegate = self;
	    [self addChildViewController:detail];
	    [detail setupView];
	    [self.view addSubview:detail.view];
	    [detail didMoveToParentViewController:self];
    }
{% endhighlight %}

如果在使用以上代码时，需要建立child View controller 和 parent view controller之间的联系。比如，如果用户选择了一个tableView里的cell，parentViewController需要知道这件事以便能够响应点击时间。所以最好的方法是table view controller定义一个协议，同时parent view controller实现这个协议。

{% highlight objc %}
@protocol DetailViewControllerDelegate
-(void)didSelectCell;
@end

@interface ParentViewController () <DetailViewControllerDelegate>
@end

@implementation ParentViewController
//....
-(void)didSelectCell
{
    //do something...
}
@end
{% endhighlight %}

虽然这样会导致view controller之间的频繁交流，但是这样保证了代码的低耦合和复用性。

---

## 分散代码
在处理tableView的时候，会有各种各样不同的，跨越model层、controller层、view层的任务。所以很有必要把这些不同的代码分散开，防止viewController成为处理这些问题的“**堆填区**”。尽可能的独立这些代码，能够使代码的可读性更好，拥有更好的可维护性与测试性。
这部分的内容可以参考[《iOS开发 简化view controller》](http://www.jianshu.com/p/cdf05c8dc3a5)，而在tableView这一章中，将会专注于如何分离view和viewController

### 消除ModelObeject和Cell之间的隔阂
在很多情况下，我们需要提交我们想要在view层展示的数据，同时我们也行维持view层和model层的分离，所以tableView中的`dateSource`常常做了超额的工作：

{% highlight objc %}
-(UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    Cell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell"];
    [cell setup];
    NSString *text = self.title;
    cell.label.text = text;
    UIImage *photo = [UIImage imageWithName:text];
    cell.photoView.image = photo;
}
{% endhighlight %}

这样`dataSorce`会变得很杂乱，应该将这些东西分到cell的category中。

{% highlight objc %}
@implementation Cell (ConfigText)

-(void)configCellWithTitle:(NSString *)title
{
    self.label.text = title;
    UIImage *photo = [UIImage imageWithName:title];
    cell.photoView.image = photo;
    return cell;
}
{% endhighlight %}

这样的话`dataSource`将会变得十分简单。

{% highlight objc %}
-(UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    Cell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell"];
    [cell configCellWithTitle:self.title];
    return cell;
}
{% endhighlight %}

### 复用cell
其实还可以更进一步，让同一个cell变得可以展示多种的modelObject。首先需要在cell中定义一个协议，想要在这个cell中展示的object必须遵守这个协议。然后可以修改分类中`config method`来让object来遵守这个协议，这样cell就能适应不同的数据类型。

### 在cell中处理cell状态
如果想要对tableView的行为进行设置，如选中操作后改变高光状态等，可以在tableViewController中使用委托方法：

{% highlight objc %}
-(void)tableView:(UITableView *)tableView didHighlightRowAtIndexPath:(NSIndexPath *)indexPath
{
    Cell *cell = [tableView cellForRowAtIndexPath:indexPath];
    cell.label.shadowColor = [UIColor greenColor];
}

-(void)tableView:(UITableView *)tableView didUnhighlightRowAtIndexPath:(NSIndexPath *)indexPath
{
    Cell *cell = [tableView cellForRowAtIndexPath:indexPath];
    cell.label.shadowColor = nil;
}
{% endhighlight %}

然而当想要换出这些cell或者想要重新设计的时候，仍然需要适应委托方法。cell里面的detail的实现和委托方法中对detail的实现交织在一起，所以应该将这些逻辑移到cell里面：

{% highlight objc %}
@implementation Cell
//...
-(void)setHighlighted:(BOOL)highlighted animated:(BOOL)animated
{
    [super setHighlighted:highlighted animated:animated];
    if(highlighted)
    {
	    self.label.shadowColor = [UIColor greenColor];
    }
    else
    {
	    self.label.shadowColor = nil;
    }
}
@end
{% endhighlight %}

**一个委托需要知道一个view的不同状态，但它不需要知道怎么去修改view或者有哪些属性需要设置来使这个view转变状态**，所有的逻辑应该由view来完成，而在外部只是仅仅提供一个API。这样才是view层和controller层实现代码之间的有效分离。
### 处理不同的cell类型
如果在一个tableView中有不同的cell类型，`dataSource`将会变得膨胀而难以操作，在下面的代码中，有两个不同的cell类型，一个负责展示图片和标题，另一个负责展示星标。为了分离处理不同的cell的代码，`dataSource`方法只是仅仅执行不同cell自己的设置方法。

{% highlight objc %}
-(UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    BOOL isStarRank = self.keys[(NSUInteger)indexPath.row];
    UITableViewCell *cell;
    if(isStarRank)
    {
	    cell = [self setupStarCell];
    }
    else
    {
	    cell = [self setupDefaultCell];
    }
}

-(StarCell *)setupStarCell
{
    //do something...
}

-(UITableViewCell *)setupDefaultCell
{
	//do something...
}
{% endhighlight %}

### 编辑TableView
TableView提供了方便的编辑功能，能够删除和移动cell。这些事件中，tableView的`dataSource`通过委托方法获取通知，因此经常在这些委托方法中出现对数据的修改，而修改数据很明显是model层的任务。model应该提供删除、排序等的接口，这样就能够通过`dataSource`的方法来调用。从而controller扮演了view和model之间的协调者，而不需要知道model层的实现细节。同时这样model的逻辑会变得更容易测试，因为没有混杂viewController的任务。

---
##总结
tableViewController以及其他controller应该扮演**model和view的协调者和中介者，而不应该关心属于view或者model层的任务**。谨记这点，让委托和`dataSource`变得更小和只包含公式化的代码。

这不只是减少tableViewController的体积和复杂度，同时也把域逻辑和界面逻辑放到相关的类中，把实现细节包裹在简单的API接口中，最终提高了代码的可读性和代码的协调能力。
