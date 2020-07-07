---
layout:     post
title:      __autoreleasing & autoreleasePool
subtitle:   
date:       2019-02-19
author:     Maqy
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - iOS
---

### __autoreleasing

```
int main() {
    __weak NSObject * ob = nil;
    __weak NSObject * tb = nil;
    {
        __autoreleasing NSObject *ooo = [NSObject new];
        ob = ooo;

        NSObject *ttt = [NSObject new];
        tb = ttt;
    }
    NSLog(@"%@", ob);
    NSLog(@"%@", tb);
}
```

打印结果如下：

```
2020-07-06 19:51:47.146624+0800 MMM[54712:5319696] <NSObject: 0x60000052c260>
2020-07-06 19:51:47.146743+0800 MMM[54712:5319696] (null)
```

可见加了__autoreleasing延迟释放了

__autoreleasing的作用是将所修饰的对象加入到最近的自动释放池中，并跟随着自动释放池的释放而释放，当代码中的花括号生命周期结束的时候，`ooo`和`ttt`的对象都会引用计数减一，但是因为`ooo`加入到了自动释放池，所以还是有引用的，导致在花括号生命周期结束的时候并没有释放，而`ttt`则随着花括号的生命周期结束而结束了。

原理稍后再说。

### @autoreleasepool

其作用是将pool花括号括起来的对象的引用计数加一，在花括号结束的时候减一，来达到批量释放的目的。

其常用的场景如下：

```
for (NSInteger i = 0; i < 10000000; i++) {
    @autoreleasepool {
        NSString *str = [NSString stringWithFormat:@"hello -%04d", i];
        NSLog(@"%@", str);
    }
}
```

如上代码，如果加了`@autoreleasepool`那么内存会保持平稳，而如果不加的话，内存会慢慢的飙升。

这里使用了`NSLog(@"%@", str)`，导致运行起来速率比较慢，所以我们可以看到内存会慢慢的升上去，如果只是创建的话，内存会飙升。

那如果我们for循环外部使用了array持有对象内？

```
NSMutableArray *arr = @[].mutableCopy;
for (NSInteger i = 0; i < 10000000; i++) {
    @autoreleasepool {
        NSString *str = [NSString stringWithFormat:@"hello -%04d", i];
        [arr addObject:str];
    }
}
```

答案是内存也会飙升，因为这里加不加`@autoreleasepool`其实并没有什么用

### 原理概述

简单来讲，其实 autoreleasepool 是个链表结构，而且每个链表存储的内存大小是固定的，pool与pool之间是双向链表的关系，每次释放的时候就是拿到当前栈顶的pool，然后将当前pool的对象逐个释放，再出栈

而我们的app是有一个runloop的，那么runloop每个循环都用一个pool包裹了一次，这样其实我们加了`__autoreleasing`的变量其实是加到了栈顶的pool里，然后跟随栈顶的pool的生命周期。

### 原理探究

下面我们看下代码：

```
#import <Foundation/Foundation.h>

int main(int argc, char * argv[]) {
    NSString * appDelegateClassName;
    @autoreleasepool {
        NSObject *ob = [NSObject new];
        // Setup code that might create autoreleased objects goes here.
        NSLog(@"lalala");
    }
    return 0;
}
```

通过Clang转译成c++代码，在目录里输入命令`clang -rewrite-objc main.m`，即可看到一个`main.cpp`的文件的生成，打开可以看到转化后的代码

```
... 省略 ...
extern "C" __declspec(dllimport) void * objc_autoreleasePoolPush(void);
extern "C" __declspec(dllimport) void objc_autoreleasePoolPop(void *);

struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};

... 省略 ...

int main(int argc, char * argv[]) {
    NSString * appDelegateClassName;
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        NSObject *ob = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new"));

        NSLog((NSString *)&__NSConstantStringImpl__var_folders_6l_nl0z4nk11xjgqq_fs490l9kw0000gp_T_main_3ae0c9_mi_0);
    }
    return 0;
}
static struct IMAGE_INFO { unsigned version; unsigned flag; } _OBJC_IMAGE_INFO = { 0, 2 };
```

可以看到 `@autoreleasepool`被转译成了 `__AtAutoreleasePool __autoreleasepool;`，它的生命周期就是花括号里面，而`__AtAutoreleasePool __autoreleasepool;`对象的构造和析构函数里分别执行了`objc_autoreleasePoolPush();`和`objc_autoreleasePoolPop(atautoreleasepoolobj)`，也就是说花括号开始的时候，我们压栈了一个pool，结束的时候出栈了一个pool，也就是说pool所包裹起来的对象的引用计数在开始的时候会+1，pool出栈的时候统一-1。

简单来说pool的操作就是 压栈，添加对象引用，出栈。那么接下来看下这个数据结构是怎么定义的吧～

#### AutoreleasePoolPage

```
class AutoreleasePoolPage 
{

#define POOL_SENTINEL nil
    static pthread_key_t const key = AUTORELEASE_POOL_KEY;
    static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
    
    // 这个SIZE是固定大小的，也就是说每个page在创建的时候所分配的内存大小是一样的
    static size_t const SIZE = 
#if PROTECT_AUTORELEASEPOOL
        PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif
    static size_t const COUNT = SIZE / sizeof(id);

    magic_t const magic;
    // 这个是所持有的变量的数组，是一块连续的内存区，其操作可以前后移动，可以认为是双向链表
    id *next;
    // 所在线程
    pthread_t const thread;
    // 双向链表，page存满了就会新建一个
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    // 链表深度
    uint32_t const depth;
    uint32_t hiwat;
    
    static void * operator new(size_t size) {
        // 创建的page的内存大小是固定的
        return malloc_zone_memalign(malloc_default_zone(), SIZE, SIZE);
    }
    static void operator delete(void * p) {
        return free(p);
    }
}
```

#### push

```
static inline void *push() 
{
    id *dest;
    if (DebugPoolAllocation) {
        // Each autorelease pool starts on a new pool page.
        dest = autoreleaseNewPage(POOL_SENTINEL);
    } else {
        dest = autoreleaseFast(POOL_SENTINEL);
    }
    assert(*dest == POOL_SENTINEL);
    return dest;
}
// 其实push的主要操作就是压入一个哨兵对象，用于标记起始位置
static inline id *autoreleaseFast(id obj)
{
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);
    } else if (page) {
        return autoreleaseFullPage(obj, page);
    } else {
        return autoreleaseNoPage(obj);
    }
}
```

#### pop

Pop主要调用了这个方法，其实就是

```
void releaseUntil(id *stop) 
    {
        // Not recursive: we don't want to blow out the stack 
        // if a thread accumulates a stupendous amount of garbage
        
        while (this->next != stop) {
            // 获取栈顶page
            AutoreleasePoolPage *page = hotPage();

            // 如果page是空的，移动链表到上一个page，并设置成hotpage
            while (page->empty()) {
                page = page->parent;
                setHotPage(page);
            }

            page->unprotect();
            // 获取补货的对象
            id obj = *--page->next;
            memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
            page->protect();

            if (obj != POOL_SENTINEL) {
                objc_release(obj); // 释放对象
            }
        }

        setHotPage(this);
    }
```

#### add

```
id *add(id obj)
{
    assert(!full());
    unprotect();
    id *ret = next;  // faster than `return next-1` because of aliasing
    *next++ = obj;
    protect();
    return ret;
}
```

add就比较简单了，直接存储对象地址



### 总结

1.当我们用 autoreleasePool 的时候并不一定会创建一个 Page来存储，也可能是上次没有存满的

2.pool的内存大小是固定的，且pool是存储在全局数据区的

3.每个线程都有它自己的pool，线程间不能共用pool

4.Page的真实结构就是一个双向链表，释放的时候不以Page为单位，而是以哨兵对象为标记，其实就是一个空标记