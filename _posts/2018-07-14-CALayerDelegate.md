---
layout:     post
title:      CALayerDelegate
subtitle:   
date:       2018-07-14
author:     Maqy
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - iOS
---



### 简介

CALayerDelegate 是一个非正式协议，且所有方法都可选实现，只有当你的view需要重绘时，才会调用。

它的主要功能是：当你的view需要绘制时，提供一系列的绘制流程的控制，来让你自定义实现你的绘制。



### 正文

#### 绘制相关

```
// 当layer被重绘时调用(如果实现了)，可以设置contents
optional public func display(_ layer: CALayer)
// 当实现了drawRect，且没实现display时调用
optional public func draw(_ layer: CALayer, in ctx: CGContext)
```

比如我实现了一个`CustomView`继承自`UIView`:

```
class CustomView: UIView {

	override public func display(_ layer: CALayer) {
        NSLog("display(_ layer: CALayer) is called")
    }
    
    override public func draw(_ layer: CALayer, in ctx: CGContext) {
        NSLog("draw(_ layer: CALayer, in ctx: CGContext) is called")
        super .draw(layer, in: ctx)
    }
    
    override public func draw(_ rect: CGRect) {
        NSLog("draw(_ rect: CGRect) is called")
    }

}
```

1. 当我们创建并添加该view后，实际上这三个方法都不会被调用，只有当我们调用`view.setNeedsDisplay() `或`view.layer.display() `时才会被调用

2. 当我们实现了上述三个方法后，那么输出结果为`display(_ layer: CALayer) is called`，证明如果实现了第一个方法，那么不管我们实不实现`drawRect`都不会继续向下调用

3. 当我们不实现`display(_ layer: CALayer)`，如果我们没有实现`drawRect`，那么输出结果为空，证明如果系统检测不到`drawRect`的实现，那么不会调用`draw(_ layer: CALayer, in ctx: CGContext)`

4. 当我们不实现`display(_ layer: CALayer)`，如果我们实现了`drawRect`，那么输出结果为

   ```
   draw(_ layer: CALayer, in ctx: CGContext) is called
   draw(_ rect: CGRect) is called
   ```

   证明如果系统检测到`drawRect`的实现，那么会实现一个默认的`draw(_ layer: CALayer, in ctx: CGContext)`，在这个里面会创建一个空白的寄宿图属性，其大小尺寸为`layer.contentsScale`乘以示图大小

#### 动画相关

```
	/* If defined, called by the default implementation of the
     * -actionForKey: method. Should return an object implementating the
     * CAAction protocol. May return 'nil' if the delegate doesn't specify
     * a behavior for the current event. Returning the null object (i.e.
     * '[NSNull null]') explicitly forces no further search. (I.e. the
     * +defaultActionForKey: method will not be called.) */
    
    // 为layer提供一个默认的动画
    @available(iOS 2.0, *)
    optional public func action(for layer: CALayer, forKey event: String) -> CAAction?
```



一个layer的动画的调用顺序如下：

1.如果实现了该delegate，则会调用`action(for layer: CALayer, forKey event: String) -> CAAction?`

如果返回nil，则会继续向下查找相应的动画；

如果返回NSNULL()，则会没有动画；

如果返回一个CAAction，则会按照返回的定义来展现动画。

2.如果继续向下查找则会找到layer的`actions`，来寻找相应的动画

3.如果还没有会继续找style字典，来寻找动画

3.如果还没有，那么会调用` defaultAction(forKey:)`一个系统默认的动画



如何修改隐式动画的时间呢？

```
CATransaction.begin()
CATransaction.setAnimationDuration(4)
self.sublayer.backgroundColor = UIColor.blue.cgColor
CATransaction.commit()
```



### 总结

CALayerDelegate主要提供了一些重绘，布局和动画相关的东西，并能够提供给我们一些自定义的操作。
