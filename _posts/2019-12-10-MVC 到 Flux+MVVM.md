---
layout:     post
title:      MVC 到 Flux+MVVM
subtitle:   
date:       2019-12-10
author:     Maqy
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - frames
---



### MVC

传统的MVC模式主要分为三层：Model，View，Controller

Model: 数据层

View: 渲染层

Controller: 逻辑层

曾经移动端最传统的模式MVC，将数据，逻辑和Ui层粉的很清晰明确，对于一些业务不是特别复杂的情况呢，这个模式还是够用的，但对于稍微复杂的模式来说，就很头疼了。最明显的缺点就是controller层越来越臃肿，稍好的情况呢，我们会抽取一些handler或caterage之类的角色来稍微分一部分逻辑出去，但依然不能做到很好的隔离。

### 背景

尤其是对于直播这种业务情况，一个直播VC内部有那么多的业务，那么多的控件，各种业务之间有很复杂的数据流向，每个业务之间还有耦合，可以想像这里有那么多的坑，那么要怎么才能更好的玩起来呢？

其实对于直播这种业务，我们主要为了解决下面这几个大问题：

1.数据流向，避免造成图状的数据流向，难以排查问题

2.解决不同业务之间的耦合

3.单个业务要容易扩展，修改的时候不去影响其他的业务

其实就是怎么做到设计模式上的五大原则。

### Flux

为了解决数据流向问题，我们要避免每个业务都有权限去修改其他的业务，那么我们就要做成环状的数据流，也就是单向数据流，而Flux就是这样一种编程思想，简单来说他的角色分工主要是下图所示：

这里是官网的Flux编程思想的介绍，有兴趣的可以看下哈：

http://blog.benjamin-encz.de/post/real-world-flux-ios/?utm_source=wanqu.co&utm_campaign=Wanqu+Daily&utm_medium=website

下面是数据流向图：

![](https://user-images.githubusercontent.com/12802196/86509808-37570400-be1d-11ea-97b6-962584e9e06b.png)



角色分工：

```
Store：一个场景理论上只有一个Store，它的主要作用是维护当前场景的业务流程及状态
View：根据当前的状态及时作出渲染上变更
Action：事件分两种（用户输入的或者由View层产生的事件）
Dispatcher：分发器，由Store做出反应，针对Action及时变更状态
```

数据流向：

一定要注意事件流向是单向的，避免造成图状的事件结构

下面举个例子：

首先封装出基本的Action和State：

```
// 用户产生的事件类型，或者渲染层产生的事件类型
@interface FluxAction : NSObject
@property (nonatomic, strong) NSString *fliter; // 事件内容
@end

@implementation FluxAction
+ (instancetype)actionWithFliter:(NSString *)fliter {
    FluxAction *act = [FluxAction new];
    act.fliter = fliter;
    return act;
}
@end


// 记录app运行中的状态，会根据事件产生变更
@interface FluxState : NSObject
@property (nonatomic, strong) NSString *fliter;
@end

@implementation FluxState
- (FluxState *)producter {
    NSLog(@"%@ call %@ method fliter %@", NSStringFromClass([self class]), NSStringFromSelector(_cmd), self.fliter);
    return self;
}
@end
```

接下来封装下Store层：

```
@interface FluxStore : NSObject
@property (nonatomic, strong) FluxState *state;
@end

@implementation FluxStore

- (instancetype)init {
    self = [super init];
    self.state = [FluxState new];
    return self;
}

- (void)dispatchAction:(FluxAction *)action {
    NSLog(@"%@ call %@ method", NSStringFromClass([self class]), NSStringFromSelector(_cmd));
    self.state.fliter = action.fliter;
    self.state = [self.state producter];
}

@end
```

这里只是写了个dispatch的方法，没有写分发器，当收到事件分发的时候，内部状态会根据action及时作出变更，那么接下来使用的时候，渲染层是需要借助KVO来进行变更的。

使用如下：

```
- (void)executor {
    self.store = [FluxStore new];
    [self.store addObserver:self forKeyPath:@"state" options:NSKeyValueObservingOptionNew context:nil];
    [self.store dispatchAction:[FluxAction actionWithFliter:@"bug mode"]];
    [self.store dispatchAction:[FluxAction actionWithFliter:@"release mode"]];
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    NSLog(@"刷新UI fliter %@",self.store.state.fliter);
}
```

这样在产生事件的时候，store会立刻做出反应，然后渲染层会根据state的变更立刻刷新自己的状态，从数据流向上看是单向的，之后在调试时我们也可以轻松的知道action是那里产生的，从而很容易的调试，项目结构也很容易理解。

### 应用

那么在直播中是怎么应用的呢？

实际上就是把直播的流程抽象成一个Store，里面维护直播的关键状态的state，当state变更的时候，会将state分发到所有的业务组件中，譬如PK，连麦，聊聊，排行榜等组件(我们简称plugin)。

那么为了避免plugin之间胡乱的数据流向，我们要做到的理想的数据流向是：store->plugin->store

但这样还是有问题的，比如我们肯定不能把所有的plugin写成一个啊，每个小业务都很复杂，如果都写到一起，那就成为一个庞然大物了，为了避免这个问题呢，我们需要在Flux的基础上引入MVVM模式。

那在直播间里就需要把View层替换成MVVM模式，也就是说每个plugin都要采用MVVM或MVC的模式来实现。

### MVVM

所谓的MVVM其实是从MVC演变过来的，MVVM在MVC的基础上多了一层ViewModel

这里就简单说下ViewModel的逻辑吧：

ViewModel主要是将Controller的和model有关的逻辑迁移了过来，我们最普遍的就是将网络层或一些model的计算逻辑迁移进ViewModel，从而避免Controller过于臃肿，迁移出来后，Controller更类似于一个组装者，来负责各层之间的协调工作。

![](https://upload-images.jianshu.io/upload_images/2987980-e73563f01a332d9b.png)



### Question

那么用Flux+MVVM就完全解决了这种特别复杂的业务么？

肯定不行啊，各个plugin之间还是需要做交互的，那么像这种通常几十个业务间的耦合怎么处理呢？

这里就需要使用DI（依赖注入）来解决啦，可以看下DI的文章哈！！！

