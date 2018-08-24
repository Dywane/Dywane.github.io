---
layout: post
title: "DWAnimatedLabel"
date: 2018-08-24
project: true
excerpt: "一个`UILabel`的子类，添加各类型动画"
tag:
- iOS开发
- Github
comments: true
---

一个`UILabel`的子类，添加各类型动画，灵感来自[RQShineLabel](https://github.com/zipme/RQShineLabel).

![wave](https://raw.githubusercontent.com/Dywane/DWAnimatedLabel/master/Gif/wave.gif)

[中文教程](https://dywane.github.io/使用CADisplayLink实现UILabel动画特效/)

## 功能
-  `UILabel`子类，便于使用
- 使用 `CADisplayLink`来展示流畅的动画
- 四种不同的动画类型
- 纯Swift语言编写

![typewritter](https://raw.githubusercontent.com/Dywane/DWAnimatedLabel/master/Gif/typewriter.gif)

![shine](https://raw.githubusercontent.com/Dywane/DWAnimatedLabel/master/Gif/shine.gif)

![fade](https://raw.githubusercontent.com/Dywane/DWAnimatedLabel/master/Gif/fade.gif)

## 安装

在 Podfile中加入

```ruby
target 'MyApp' do
  pod 'DWAnimatedLabel', '~> 1.1'
end
```

然后在终端运行 `pod install`

另外也可以使用 `pod try DWAnimatedLabel` 来做一个测试运行

## 用法

你需要先 import 这个模块

```swift 
import DWAnimatedLabel
```

然后像创建普通的 `UILabel`一样创建这个特殊的 Label

```swift
let label = DWAnimatedLabel(frame: CGRect(x: 20, y: 44, width: UIScreen.main.bounds.size.width, height: 100))
label.text = "LOADING"
label.font = UIFont.systemFont(ofSize: 70, weight: .bold)
```

你可以通过 `animationType` 属性来选择动画类型

```swift
label.animationType = .wave
```

如果你在使用 `wave`动画，你还需要设置`placeHolderColor` 属性来定义底部颜色，否则他会使用 `UIColor.lightGray`作为背景色。

```swift
label.placeHolderColor = .blue
```

设置完成后你可以使用 `startAnimation(duration: TimeInterval, _ completion:(() -> Void)?)` 来展示动画

## Requirements

- iOS 9.0 +
- Swift 4
- Xcode 9

## Contribution

欢迎对项目提供宝贵意见，有问题也可以在[Github Issue](https://github.com/Dywane/DWAnimatedLabel/issues)上与我进行联系

## License

DWAnimatedLabel is open-sourced software lincened under the MIT license.

## Credits

有兴趣可以关注[我的博客](https://dywane.github.io)，里面有更多内容
