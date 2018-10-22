 日常开发中，我们经常会注册一些通知、发起一些请求，当我们不需要时应及时注销通知，取消掉请求。否则，就有可能产生问题或者崩溃。比如我们会在控制器的viewDidLoad里面注册一些通知，然后在dealloc里面注销掉通知。或者当我们退出控制器时，将所有的当前发起的请求都Cancel掉。这在MRC开发下是非常常见的，因为请求返回时，回调代理时可能为野指针。这种手动注销的方式有些繁琐，开发中经常会遗忘导致问题被隐藏起来。正因为如此，我们希望可以提供一种架构去自动解决此类注销通知、取消请求的调用。

    分析这个场景可以发现，这些通知的观察者以及请求，它们的生命周期是依赖于某个宿主的。当某个宿主对象销毁时，观察者以及请求也会被销毁。也就是自动调用注销的时机确定在对象释放时dealloc。同时，我们不应该对宿主的类型有特殊的要求，宿主可能是控制器，某个自定义控件，或者数据模块，即宿主要求是NSObject类型的对象。另外，我们希望可以自定义注销时的操作，比如是注销事件观察，或者是取消请求。以上是对这个架构最基本的要求，此外我们还有一些更高的要求，希望架构对性能的影响尽可能的小，避免产生一些循环引用的问题，使用要简单，最好有一些管理功能。

    相对于其他需求点，捕获NSObject对象释放的时机是最为关键的需求。通常我们会想到使用Method Swizzling的方式，替换NSObject的dealloc方法。但这里有几个问题需要注意。

    1、ARC开发下，dealloc作为关键字，编译器是有所限制的。会产生编译错误“ARC forbids use of 'dealloc' in a @selector”。不过我们可以用运行时的方式进行解决。

    2、dealloc作为最为基础，调用次数最为频繁的方法之一。如对此方法进行替换，一是代码的引入对工程影响范围太大，二是执行的代价较大。因为大多数dealloc操作是不需要引入自动注销的，为了少数需求而对所有的执行都做修正是不适当的。

    所以Method Swizzling可以作为最后的备选方案，但不适合作为首选方案。

    另外一个思路是在宿主释放过程中嵌入我们自己的对象，使得宿主释放时顺带将我们的对象一起释放掉，从而获取dealloc的时机点。显然AssociatedObject是我们想要的方案。相比Method Swizzling方案，AssociatedObject方案的对工程的影响范围小，而且只有使用自动注销的对象才会产生代价。

鉴于以上对比，于是采用构建一个释放通知对象，通过AssociatedObject方式连接到宿主对象，在宿主释放时进行回调，完成注销动作。

    下面是释放通知类RFDestoryNotify的声明。通过rfDestoryNotifySetName方法对宿主对象添加销毁监听，通过block回调具体的注销方法。 

![img](http://nos.netease.com/knowledge/2860d0e4-0f88-4193-8383-bf05504ab34a)

    使用RFDestoryNotify之前需要对整个释放过程的时序特性有所了解，才能写出合适的代码。所以，我们先编写如下的代码进行调试。

![img](http://nos.netease.com/knowledge/237e6bdd-02ec-4d78-8fbf-c6ea0a67de15)

    我们在宿主的释放函数dealloc（33），释放监听回调（58），宿主释放（61），宿主释放后（62）添加四个回调。经过调试，其触发时序如下： 

![img](http://nos.netease.com/knowledge/74eb45bf-3af7-489e-ae18-aee64b1aed80)

    整个释放过程是同步的，先进行宿主s_owner的释放，再释放关联的RFDestoryNotify对象，然后再触发回调RFDestoryNotifyBlock，完成整个过程。

    从时序看，回调RFDestoryNotifyBlock在宿主s_owner的释放动作之后，但宿主s_owner依然存活，由此可以知道宿主s_owner所持有的对象成员已全部执行过release，但基本类型的成员值还保留着。从调试结果看，实际也是这样。

    因此，我们可以知道在释放回调时，宿主所持有的对象成员是不可靠的，但宿主的内存地址是可靠的。这也是我们对RFDestoryNotify添加userInfo的原因。

以上是调试得到的结果，我们最好还是从OC的源代码层面加以确认。 

![img](http://nos.netease.com/knowledge/06765f62-2387-4fa3-8770-5756c538deb6)

    上面是OC对象销毁时的处理，查找资料后，发现object_cxxDestruct负责遍历持有的对象，并进行析构销毁。_object_remove_assocations负责销毁关联对象的销毁。clearDeallocating负责对weak持有本对象的引用置nil。从代码上看，代码执行时序与调试时的时序也是一致的。这也证明了，上面描述的RFDestoryNotify的运行特性是正确的，可靠的。

    附：

    ARC下dealloc过程及.cxx_destruct的探究

    http://blog.sunnyxx.com/2014/04/02/objc_dig_arc_dealloc/

    目前在我们的APP项目中，这项设计已经普遍得到应用。通过引入RFEvent，实现了观察者释放时，可以自动注销相对应的事件观察。在网络事务请求Work中引入这一设计，可以声明全局、宿主两种不同的生命周期。方便对网络请求进行管理，增强了程序的健壮性。在自定义dialog中引入这一设计，可以在回退时，方便的对宿主创建的dialog进行销毁，为页面自动跳转提供便利。从实践效果看，对开发效率的提升和代码质量的提升还是很明显的，因此推荐大家，希望对大家日后的开发工作起到积极的作用。

    附相关代码参考

    RFDestoryNotify

    https://github.com/refusebt/RFDestoryNotify

    RFEvent

    https://github.com/refusebt/RFEvent