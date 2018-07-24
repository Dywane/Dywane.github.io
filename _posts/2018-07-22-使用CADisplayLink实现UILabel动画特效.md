---
layout: post
title: "使用CADisplayLink实现UILabel动画特效"
date: 2018-07-22
excerpt: "在开发时，我们有时候会遇到需要定时对UIView进行重绘的需求，进而让view产生不同的动画效果。"
tag:
- iOS开发
- 教程
comments: true
---

> 在开发时，我们有时候会遇到需要定时对UIView进行重绘的需求，进而让view产生不同的动画效果。

**本文[项目](https://github.com/Dywane/DWAnimatedLabel/tree/master)**

#### 效果图 
![typewritter](https://raw.githubusercontent.com/Dywane/DWAnimatedLabel/master/Gif/typewriter.gif)

![shine](https://raw.githubusercontent.com/Dywane/DWAnimatedLabel/master/Gif/shine.gif)

![fade](https://raw.githubusercontent.com/Dywane/DWAnimatedLabel/master/Gif/fade.gif)

![wave](https://raw.githubusercontent.com/Dywane/DWAnimatedLabel/master/Gif/wave.gif)

## 初探 CADisplayLink

定时对View进行定时重绘可能会第一时间想到使用`NSTimer`，但是这样的动画实现起来是**不流畅**的，因为在timer所处的`runloop`中要处理多种不同的输入，导致timer的最小周期是在50到100毫秒之间，一秒钟之内最多只能跑20次左右。

但如果我们希望在屏幕上看到流畅的动画，我们就要维持60帧的刷新频率，也就意味着每一帧的间隔要在**0.016**秒左右，`NSTimer`是无法实现的。所以要用到`Core Animation`的另一个timer，`CADisplayLink`。

在`CADisplayLink`的头文件中，我们可以看到它的使用方法跟`NSTimer`是十分类似的，其同样也是需要注册到RunLoop中，但不同于`NSTimer`的是，它在**屏幕需要进行重绘时**就会让RunLoop调用`CADisplayLink`指定的selector，用于准备下一帧显示的数据。而`NSTimer`是需要在**上一次RunLoop整个完成之后**才会调用制定的selector，所以在调用频率与上比`NSTimer`要频繁得多。

另外和`NSTimer`不同的是，`NSTimer`可以指定`timeInterval`，对应的是selector调用的间隔，但如果`NSTimer`触发的时间到了，而RunLoop处于阻塞状态，其触发时间就会**推迟**到下一个RunLoop。而`CADisplayLink`的timer间隔是**不能调整**的，固定就是一秒钟发生**60**次，不过可以通过设置其`frameInterval`属性，设置调用一次selector之间的**间隔帧数**。另外需要注意的是如果selector执行的代码超过了`frameInterval`的持续时间，那么`CADisplayLink`就会**直接忽略**这一帧，在下一次的更新时候再接着运行。


## 配置 RunLoop

在创建CADisplayLink的时候，我们需要指定一个RunLoop和`RunLoopMode`，通常RunLoop我们都是选择使用主线程的RunLoop，因为所有UI更新的操作都必须放到主线程来完成，而在模式的选择就可以用`NSDefaultRunLoopMode`，但是不能保证动画平滑的运行，所以就可以用`NSRunLoopCommonModes`来替代。但是要小心，因为如果动画在一个**高帧率**情况下运行，会导致一些别的类似于定时器的任务或者类似于滑动的其他iOS动画会暂停，直到动画结束。

```swift
private func setup() {
	_displayLink = CADisplayLink(target: self, selector: #selector(update))
	_displayLink?.isPaused = true
	_displayLink?.add(to: RunLoop.main, forMode: .commonModes)
}

```

## 实现不同的字符变换动画

在成功建立`CADisplayLink`计时器后，就可以着手对字符串进行各类动画操作了。在这里我们会使用`NSAttributedString`来实现效果

在`setupAnimatedText(from labelText: String?)`这个方法中，我们需要使用到两个数组，一个是`durationArray`，一个是`delayArray`，通过配置这两个数组中的数值，我们可以实现对字符串中各个字符的**出现时间**和**出现时长**的控制。

### 打字机效果的配置

- 每个字符出现所需时间相同
- 下一个字符等待上一个字符出现完成后再出现
- 通过修改`NSAttributedStringKey.baselineOffset`调整**字符位置**

```swift
case .typewriter:
	attributedString.addAttribute(.baselineOffset, value: -label.font.lineHeight, range: NSRange(location: 0, length: attributedString.length))
	let displayInterval = duration / TimeInterval(attributedString.length)
	for index in 0..<attributedString.length {
		durationArray.append(displayInterval)
		delayArray.append(TimeInterval(index) * displayInterval)
	}

```

### 闪烁效果的配置

- 每个字符出现所需时间随机
- 确保所有字符能够在`duration`内均完成出现
- 修改`NSAttributedStringKey.foregroundColor`的**透明度**来实现字符的出现效果

```swift
case .shine:
	attributedString.addAttribute(.foregroundColor, value: label.textColor.withAlphaComponent(0), range: NSRange(location: 0, length: attributedString.length))
	for index in 0..<attributedString.length {
		delayArray.append(TimeInterval(arc4random_uniform(UInt32(duration) / 2 * 100) / 100))
		let remain = duration - Double(delayArray[index])
		durationArray.append(TimeInterval(arc4random_uniform(UInt32(remain) * 100) / 100))
	}
```

### 渐现效果的配置

- 每个字符出现所需时间渐减
- 修改`NSAttributedStringKey.foregroundColor`的**透明度**来实现字符的出现效果

```swift
case .fade:
	attributedString.addAttribute(.foregroundColor, value: label.textColor.withAlphaComponent(0), range: NSRange(location: 0, length: attributedString.length))
	let displayInterval = duration / TimeInterval(attributedString.length)
	for index in 0..<attributedString.length  {
		delayArray.append(TimeInterval(index) * displayInterval)
		durationArray.append(duration - delayArray[index])
	}
```

## 完善每一帧的字符串更新效果

接下来就需要完善刚才在`CADisplayLink`中配置的`update`方法了，在这个方法中我们会根据我们刚才配置的两个数组中的相关数据对字符串进行变换。

### 核心代码

- 通过**开始时间**与**当前时间**获取**动画进度**
- 根据**字符位置**对应`duationArray`与`delayArray`中的数据
- 根据`durationArray`与`delayArray`中的数据计算当前字符的**显示进度**

```swift
var percent = (CGFloat(currentTime - beginTime) - CGFloat(delayArray[index])) / CGFloat(durationArray[index])
percent = fmax(0.0, percent)
percent = fmin(1.0, percent)
attributedString.addAttribute(.baselineOffset, value: (percent - 1) * label!.font.lineHeight, range: range)
```
随后便可以将处理完的`NSAttributedString`返回给label进行更新

## 番外：利用正弦函数实现波纹进度

### 波纹路径

首先介绍一下正弦函数：`y = A * sin(ax + b)`

- 在 x 轴方向平移 b 个单位（左加右减）
- 横坐标伸长（0 < a < 1）或者缩短（a > 1) 1/a 倍
- 纵坐标伸长（A > 1）或者缩短（0 < A < 1）A 倍

在简单了解了这些知识后，我们回到`wavePath()`方法中，在这个方法我们使用正弦函数来绘制一段`UIBezierPath`：

```swift
let originY = (label.bounds.size.height + label.font.lineHeight) / 2
let path = UIBezierPath()
path.move(to: CGPoint(x: 0, y: _waveHeight!))
var yPosition = 0.0
for xPosition in 0..<Int(label.bounds.size.width) {
	yPosition = _zoom! * sin(Double(xPosition) / 180.0 * Double.pi - 4 * _translate! / Double.pi) * 5 + _waveHeight!
	path.addLine(to: CGPoint(x: Double(xPosition), y: yPosition))
}
path.addLine(to: CGPoint(x: label.bounds.size.width, y: originY))
path.addLine(to: CGPoint(x: 0, y: originY))
path.addLine(to: CGPoint(x: 0, y: _waveHeight!))
path.close()
```

### 波纹高度与动画的更新

- 随着进度高度不断升高
- 随着进度波纹不断波动

在`CADisplayLink`注册的`update`的方法中，我们对承载了波纹路径的Layer进行更新

```swift
_waveHeight! -= duration / Double(label!.font.lineHeight)
_translate! += 0.1
if !_reverse {
	_zoom! += 0.02
	if _zoom! >= 1.2 {
		_reverse = true
	}
} else {
	_zoom! -= 0.02
	if _zoom! <= 1.0 {
		_reverse = false
	}
}
shapeLayer.path = wavePath()
```

## 结语

以上就是我对`CADisplayLink`的一些运用，其实它的使用方法还有很多，可以利用它实现更多更复杂而精美的动画，同时希望各位如果有更好的改进也能与我分享。

如果你喜欢这个[项目](https://github.com/Dywane/DWAnimatedLabel/tree/master)，欢迎到GitHub上给我一个star。

#### 参考
- [RQShineLabel](https://github.com/zipme/RQShineLabel)
- [Apple Developer Document - CADisplayLink](https://developer.apple.com/documentation/quartzcore/cadisplaylink)
- [iOS核心动画高级技巧](https://zsisme.gitbooks.io/ios-/content/chapter11/frame-timing.html)
