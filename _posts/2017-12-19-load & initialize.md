---
layout:     post
title:      load & initialize
subtitle:   
date:       2017-12-19
author:     Maqy
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - iOS
---

NSObject 的分类里有两个比较常用的runtime方法

```
+ (void)load;
+ (void)initialize; 
```

方法对比：

1.load是main前调用，initialize是main后，第一次初始调用对象方法的时候调用

2.load不参与继承体系，initialize参与继承体系，也就是说子类会覆盖父类，分类会覆盖当前类

3.对于同一个类来说，两者都只调用一次



### 实践

```
/// 父类
@interface LoadFather : NSObject
@end
@implementation LoadFather
+ (void)load {
    NSLog(@"LoadFather Load");
}
+ (void)initialize {
    NSLog(@"LoadFather initialize");
}
@end

/// 子类
@interface LoadChild : LoadFather
@end
@implementation LoadChild
@end

/// 子子类
@interface LoadChildChild : LoadChild
@end
@implementation LoadChildChild
+ (void)load {
    NSLog(@"LoadChildChild Load");
}
+ (void)initialize {
    NSLog(@"LoadChildChild initialize");
}
@end

/// 分类
@interface LoadChildChild (Category)
@end
@implementation LoadChildChild (Category)
+ (void)load {
    NSLog(@"LoadChildChild Category Load");
}
+ (void)initialize {
    NSLog(@"LoadChildChild Category initialize");
}
@end
```



如上定义好继承关系后，执行 `  [LoadChildChild new];`得到如下输出：

```
2020-05-24 23:53:33.107923+0800 test[56356:1702538] LoadFather Load
2020-05-24 23:53:33.108433+0800 test[56356:1702538] LoadChildChild Load
2020-05-24 23:53:33.108522+0800 test[56356:1702538] LoadChildChild Category Load
2020-05-24 23:53:33.168233+0800 test[56356:1702538] LoadFather initialize
2020-05-24 23:53:33.168350+0800 test[56356:1702538] LoadFather initialize
2020-05-24 23:53:33.168447+0800 test[56356:1702538] LoadChildChild Category initialize
```

#### 结论1

可以看到，load的调用顺序是 先父类--->再子类--->再分类的关系。

-------

那么父类的分类会先于子类调用吗？答案是不会。

为什么呢？

因为这个和编译器的编译顺序有关，编译器在链接过程中，会先链接父类，再链接子类，最后链接分类，不管是谁的分类都是最后链接的。

--------

其实这个方法的同类型的加载顺序是可以微调的，在 `compile source`里调整文件顺序可以进行微调，但怎么调也不能讲子类调到父类前，分类调到类前



#### 结论2

initialize 方法是参与继承体系的，也就是会覆盖调用

------

所以使用该方法的时候，官方文档也明确指出需要在方法里判断

```
+ (void)initialize {
  if (self == [ClassName self]) {
    // ... do the initialization ...
  }
}
```

这里为什么要判断类型呢？

因为子类没实现的时候会调用父类的，是参与继承体系的。

假设一个常见的场景：

```
class Father {
	func initialize() {
		static var map = Int()
	}
}

class A: Father {
	func doSomeThing() {
		map.append(10)
	}
}

class B: Father {
	func doSomeThing() {
		map.append(20)
	}
}
... 
```

这里会容易产生野指针的问题，为什么呢？

因为A和B没有实现initialize方法，那么在初始调用时会调用父类的，每次都会初始一个新的map，那么如果这时候正好A或B拿到的还是原来的map的指针，但是他已经被释放了，那么就会产生 `unrecognized selector sent to instance`的崩溃

那怎么解决呢？

一个就是在initialize里加上类型判断

二个就是不要用这个方法啦



#### 结论3

最好不要用runtime的方法，如果理解不清的话容易造成很多问题

而且load还会增加启动时长

