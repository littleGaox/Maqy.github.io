平时开发的时候，我们可能都会有这样的疑问：iOS的点击事件是怎么传递的呢？hitTest有什么作用？响应链又是怎么回事？Runloop是怎么运行的呢？平时开发中也用的很少，它们又仿佛是单独存在没什么联系的。本文就想把它们都串联起来，加深我们对iOS UI中，从事件发生到事件被丢弃或被处理的整个过程的理解，以便于在某些特定的需求的时候 做一些自定义的事情。

### 摘要

因为最常见的事件就是触屏事件(点击、长按、滚动等等)，所以就以点击为例子，看看一个点击事件从发生到被丢弃或被处理 的过程。 总的来说，一个点击事件的整体流程：

1. 屏幕被触摸，主线程中的runloop被唤醒，处理点击事件
2. runloop里, 调用UIApplication，遍历application.windows里的所有window以及他们的subview递归地做hitTest，找出hitTesting view(即需要处理这个点击事件的最上层的view)
3. 给hitTesting view发点击事件，若它响应了此事件 并终止传递，则不往响应链上传递；否则，则依次往响应链上传递，直到传到UIWindow、UIApplication，都不处理就丢弃，事件传递结束。

### 1. runloop

我们在开发过程中，经常会遇到类似的需求：后台开启一个线程，这个线程不能退出，但又需要保持闲时休息，有事情可干了就唤醒处理任务，比如：

1. 自定义的网络推送
2. 后台线程监视文件变化，一旦文件变化就统计所有磁盘文件占用大小。
3. 主线程的UI交互系统

这时候就需要开启一个线程，并在线程里new一个runloop 并 run，然后再添加一些输入源到这个runloop上即可实现上述这样的功能。

下图是主线程里的runloop里的示意图

![img](http://blog.yageek.net/concurrency-objc20/images/nsrunloop.jpg)

简单的说runloop就是一个while循环，只不过在while体里没输入源到来的时候，就等着（而不是一直跑），一旦有输入源了（比如点击触屏事件、网络、文件、timer, socket），则被唤醒。它的实现的大概伪代码可以写成这样

![img](http://photo1.fanfou.com/v1/mss_3d027b52ec5a4d589e68050845611e68/ff/n0/0d/g7/tn_190514.jpg@596w_1l.jpg) ![img](http://photo2.fanfou.com/v1/mss_3d027b52ec5a4d589e68050845611e68/ff/n0/0d/g7/tj_190521.jpg@596w_1l.jpg) ![img](http://photo3.fanfou.com/v1/mss_3d027b52ec5a4d589e68050845611e68/ff/n0/0d/g7/tb_190544.jpg@596w_1l.jpg)

每个线程都可以有一个runloop，不过，主线程里的runloop是自动开启的。所以，一旦有触屏点击事件，runloop就被唤醒，开始处理这些事件:hitTest, sendEvent,结束之后又开始睡眠并等待唤醒。对于收到触屏事件的主线程runloop来说，被唤醒的第一件事就是，找到应该响应此事件的view，并把该事件先发送给这个view来处理。这个找view的过程即为hitTesting。

### 2. hitTesting

找hitTesting view的过程其实就是，找到最上层的那个能处理这个事件的view，然后把事件交给它处理。对于被触屏事件唤醒了的runloop来说，它要从application开始找，application又持有管理着所有的windows，通常所以调用堆栈和遍历顺序就是runloop -> application -> windows -> keyWindow -> subviews -> subviews -> ...

![img](http://smnh.me/images/hit-test-touch-event-flow.png)

对于从window上找到点击的view，这个过程可以描述成：从根view（即window）开始遍历，递归地从subview的最后一个(即subviews数组的反序，最后面的在最上面)开始找，若此view isUserInteractionEnabled==NO 或 hidden==YES 或 alpha <=0.01则跳过这个view，视为这个view不可点击，若不在这三个条件范围内的，并且点击区域刚好落在这个view的frame之内的，并且它没有子view或它的子view不满足条件，则视为找到。

举例来说，点击下图的绿色视图B上的B.1的test过程如下：

![img](http://smnh.me/images/hit-test-depth-first-traversal.png)

系统的UIView的hitTest的猜测实现代码:

![img](http://photo.fanfou.com/v1/mss_3d027b52ec5a4d589e68050845611e68/ff/n0/0d/g7/t4_190531.jpg@596w_1l.jpg)

流程图:

![img](http://smnh.me/images/hit-test-flowchart.png)

题外：可以利用重写hitTest做的一些事情

- 在不改变frame的前提下，扩大点击区域，这就需要重载pointInside:withEvent:
- 比如，替换UIView的hitTest，以便记录所有的ios的点击事件，简单的实现跟踪这个用户所有的点击动作，结合崩溃日志系统知道用户在崩溃之前 做了哪些动作，便于QA复现问题和开发找线索。

到此，既然已经找到了该响应点击事件的view之后，我们就差最后一步，向这个view发送事件了。

### 3. sendEvent和响应链

找到hitTesting view以后，会把view放到UITouch实例中，然后[application sendEvent:touchEvent], [window sendEvent:touchEvent] 然后顺着响应链一直传递，直到有responder对象终止了这个传递、停止了转发。

那么什么是响应链？响应链规范了iOS中所有UIResponder对象传递事件的顺序，通常一个触屏点击事件起始于hitTesting view，这个view如果是controller的view，则view的nextResponder则是controller，然后这个controller的nextResponder是它的view的superView; 若不是，则它的nextResponder则是此view的superView；以此类推，直到UIWindow，最后UIWindow的nextResponder则是UIApplication。具体响应链的关系，见官方文档里的下图：

![img](https://developer.apple.com/library/content/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Art/iOS_responder_chain_2x.png)

#### 3.1 响应链的作用

总的作用就是在所有UIResponder对象之间传递事件和发送消息，比如：

1. 在UIResponder对象之间传递点击事件
2. 方便UIView向UIViewController传递消息, 比如view加载完成，view将会显示

#### 3.2 响应链构建的时机

1. window.rootViewController = aController的时候，会在window和controller之间的建立响应链关系；
2. UIViewController之间会在 [viewController addChildViewController]的时候，会建立controller和controller之间的响应链关系。反之，若addSubview了，但是没有addChildController。此时若在controller里有[btn addTarget:controller action:@selector() forControlEvents:] 就会有问题
3. viewController和vc.view 会在loadView的时候建立响应链关系
4. view和view之间会在addSubview的时候建立响应链关系

总结：至此，从点击屏幕到事件被处理的全部过程已经完成，虽然大家可能刚上来就会写[btn addTarget:selector:controlEvent:]，但是中间的过程可能花了很久才知道，理解整个过程也许会对我们平时的开发更有帮助。