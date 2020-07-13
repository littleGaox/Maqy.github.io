---
layout:     post
title:      通过Memory来了解Runtime
subtitle:   
date:       2020-01-21
author:     Maqy
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - Runtime
---



### 前言

这里简单分享下oc的对象在内存里是怎么存储的，主要通过查看内存的各个字段的值来了解runtime究竟是怎么布局的

`memory`命令可以查看一个地址只想的内存上所存储的值，我们可以用`--help`来看参数的设置

`memory read obj`查看obj的值，读取是按照字节读取的

`x obj` 是上边的间简写

`x/x obj  -c 8 -s 8` /x可以解决大小端展示问题，-c代表展示多少个，-s代表展示的字节数

```
字节的知识在这里简单介绍下吧：
1 byte = 8 位
1 byte 的数值大小 0~255 (2 的 16 次方)
对于64位系统而言，一个指针的大小是 8 byte = 64 位
对于32位系统而言，一个指针的大小是 4 byte = 32 位
```

### Code

下面举个例子来给大家看下，runtime是如何进行内存布局的：

```
@interface ReplaceDemo : NSObject
@property (nonatomic,copy) NSString *name1;
@property (nonatomic,copy) NSString *name2;
@property (nonatomic,copy) NSString *name3;
@property (nonatomic,copy) NSString *name4;
@property (nonatomic,copy) NSString *name5;
@end

@implementation ReplaceDemo
@end

@interface ReplaceSubDemo : ReplaceDemo
@property (nonatomic,copy) NSString *name;
@end
@implementation ReplaceSubDemo
- (void)testPrint {
    NSLog(@"111222333444 %@", self.name1);
}
- (void)print {
    NSLog(@"111222333444 %@", self.name);
}
@end

- (void)TestMethod {
    ReplaceSubDemo *demo1 = [ReplaceSubDemo new];
    demo1.name = @"demo111111";
    [demo1 print];
    
    ReplaceSubDemo *demo2 = [ReplaceSubDemo new];
    demo2.name = @"demo222222";
    [demo2 print];
    
    id cls = [ReplaceSubDemo class];
    void *obj = &cls;
    [(__bridge id)obj testPrint];
    [(__bridge id)obj print];
}
```

上面的代码跑起来会怎么样呢？

```
111222333444 demo111111
111222333444 demo222222
111222333444 <ReplaceSubDemo: 0x600000a66940>
Thread 1: EXC_BAD_ACCESS (code=EXC_I386_GPFLT) 野指针
```

### 解析

首先我们用`x/x obj  -c 8 -s 8`命令分别打印 demo1, demo2, obj，其内存地址如下

```
(lldb) x/x demo1 -c 10 -s 8
0x600000a61440: 0x000000010b1493c8 0x0000000000000000
0x600000a61450: 0x0000000000000000 0x0000000000000000
0x600000a61460: 0x0000000000000000 0x0000000000000000
0x600000a61470: 0x000000010b13f698 0x0000000000000000
0x600000a61480: 0x0000bc5812bd1480 0xffffffffffff0057
(lldb) x/x demo2 -c 10 -s 8
0x600000a66940: 0x000000010b1493c8 0x0000000000000000
0x600000a66950: 0x0000000000000000 0x0000000000000000
0x600000a66960: 0x0000000000000000 0x0000000000000000
0x600000a66970: 0x000000010b13f6b8 0x0000000000000000
0x600000a66980: 0x0000bc5812bd6980 0xffffffffffff009d
(lldb) x/x obj -c 10 -s 8
0x7ffee4acd3c8: 0x000000010b1493c8 0x0000600000a66940
0x7ffee4acd3d8: 0x0000600000a61440 0x00007fff53c3c56e
0x7ffee4acd3e8: 0x0000600001d0c370 0x00007ffee4acd440
0x7ffee4acd3f8: 0x000000010b12fe45 0x00007ffee4acd46f
0x7ffee4acd408: 0x0000000000000000 0x000000010b13e338
```

可以看到每个对象的内存的首地址，也就是前八个字节的内容都是`0x000000010b1493c8`，也就是说他们都是一个类型，其调用方法的时候都是用这个地址来获取的class，也就是说对于同一种类型，其class的加载并不是在运行时，而是在链接的时候就已经分配好地址了，而且这个地址对于一个app而言，是固定的。

**Q：那么我们是怎么拿到class的对象的呢？**

```
# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
# elif __x86_64__
#   define ISA_MASK        0x00007ffffffffff8ULL
# endif

inline Class 
objc_object::ISA() 
{
    assert(!isTaggedPointer()); 
    return (Class)(isa.bits & ISA_MASK);
}
```

其实这就是我们oc里的`class`方法的实现，其就是把我们`isa`的地址和这个`ISA_MASK`做了一次按位与操作，得到的就是我们的class的实例了，其类型是`objc_class`。

所以在我们调试的时候我们可以直接用对象的首地址的值，来做一次按位与来拿到这个类的`class`

而因为我们创建的对象虽然在堆上的地址不同，但因为首地址的值相同，所以他们共用的是一个class。

**Q：怎么通过内存拿到他的superClass？**

```
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
}
struct objc_object {
private:
    isa_t isa;
}
```

首先来看一下`objc_class`和`objc_object`结构的定义吧，我只把他的属性给写了出来

我们可以看到，`objc_class`结构体的属性按排列的话，`superclass`排在第二位，结合上边的例子呢，在我们打印出对象的首地址之后，我们将其与`ISA_MASK`按位与，得到如下结果

```
// 这个就是class的地址
(lldb) p/x 0x00000001061853c8 & 0x0000000ffffffff8ULL
(unsigned long long) $1 = 0x00000001061853c8

// 打印这个对象得到的是类型
(lldb) po 0x00000001061853c8
ReplaceSubDemo

// 查看内存地址的值
(lldb) x/x 0x00000001061853c8 -c 10 -s 8  
0x1061853c8: 0x00000001061853a0 0x0000000106185378
0x1061853d8: 0x00007fd3bcc06fb0 0x000780440000000f
0x1061853e8: 0x0000600001800404 0x00007fff89e06698
0x1061853f8: 0x00007fff89e06698 0x0000600002e50190
0x106185408: 0x0001c03100000003 0x000060000182ae40

(lldb) po 0x00000001061853a0
ReplaceSubDemo

(lldb) po 0x0000000106185378
ReplaceDemo
```

按顺序输入上边的命令查看内存，可以得到这个class结构的内存布局，我们分别打印前八个字节和第二个八个字节，得到的结果分别是`ReplaceSubDemo`和`ReplaceDemo`

这里说明下这两个值为什么是`ReplaceSubDemo`和`ReplaceDemo`，这里的`ReplaceSubDemo`并不是我们刚才的class实例，而是元类的实例，也就是我们class里的isa的指针的值，而后面的`ReplaceDemo`就是他的父类的类型了，因为我们结构体的属性都是按顺序存储的，也就是你属性的排序会直接影响你这个属性在类内存里的位置，所以我们可以通过这个方法，直接在`首地址+8`的方式来获取`superclass`的实例

**Q：上面的例子里为什么一个方法能打印，另外一个就crash呢？**

这里呢就需要说下内存的布局了，我们一个对象创建好之后是怎么一个布局方式呢？主要来看下oc的创建方法的实现吧！

```
// 每个类型都有这个属性，在我们添加ivar的时候会计算这个类的内存大小，并将计算好的属性保存，但是任何对象最少的内存空间大小是16
size_t instanceSize(size_t extraBytes) {
      size_t size = alignedInstanceSize() + extraBytes;
      // CF requires all objects be at least 16 bytes.
      if (size < 16) size = 16;
      return size;
}

// 这个就简单看下就好，alloc的时候主要就是根据 instanceSize 字段来的
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
    if (checkNil && !cls) return nil;
    ... ...
    if (! cls->ISA()->hasCustomAWZ()) {
        // No alloc/allocWithZone implementation. Go straight to the allocator.
        // fixme store hasCustomAWZ in the non-meta class and 
        // add it to canAllocFast's summary
        if (cls->canAllocFast()) {
            // No ctors, raw isa, etc. Go straight to the metal.
            bool dtor = cls->hasCxxDtor();
            id obj = (id)calloc(1, cls->bits.fastInstanceSize());
            if (!obj) return callBadAllocHandler(cls);
            obj->initInstanceIsa(cls, dtor);
            return obj;
        }
        else {
            // Has ctor or raw isa or something. Use the slower path.
            id obj = class_createInstance(cls, 0);
            if (!obj) return callBadAllocHandler(cls);
            return obj;
        }
    }
    ... ...
    return [cls alloc];
}

// 这个就是内存布局的主要精髓所在了
// 
class_addIvar(Class cls, const char *name, size_t size, 
              uint8_t alignment, const char *type)
{
    // 元类是不会添加属性的
    if (cls->isMetaClass()) {
        return NO;
    }
    ......
    // fixme allocate less memory here
    
    ivar_list_t *oldlist, *newlist;
    if ((oldlist = (ivar_list_t *)cls->data()->ro->ivars)) {
        size_t oldsize = oldlist->byteSize();
        newlist = (ivar_list_t *)calloc(oldsize + oldlist->entsize(), 1);
        memcpy(newlist, oldlist, oldsize);
        free(oldlist);
    } else {
        newlist = (ivar_list_t *)calloc(sizeof(ivar_list_t), 1);
        newlist->entsizeAndFlags = (uint32_t)sizeof(ivar_t);
    }

    uint32_t offset = cls->unalignedInstanceSize();
    uint32_t alignMask = (1<<alignment)-1;
    offset = (offset + alignMask) & ~alignMask;

    ivar_t& ivar = newlist->get(newlist->count++);
#if __x86_64__
    // Deliberately over-allocate the ivar offset variable. 
    // Use calloc() to clear all 64 bits. See the note in struct ivar_t.
    ivar.offset = (int32_t *)(int64_t *)calloc(sizeof(int64_t), 1);
#else
    ivar.offset = (int32_t *)malloc(sizeof(int32_t));
#endif
    *ivar.offset = offset;   // 记录该变量对首地址的偏移
    ivar.name = name ? strdup(name) : nil;
    ivar.type = strdup(type);
    ivar.alignment_raw = alignment;
    ivar.size = (uint32_t)size; // 该变量的内存地址大小

    ro_w->ivars = newlist;
    cls->setInstanceSize((uint32_t)(offset + size));  // 将类的大小重置并加上变量的内存大小
    return YES;
}
```

我将内存开辟的过程简单叙述一下就是：没当我们给类添加变量的时候会记录下该变量在类内的地址偏移，并计算好加上该属性之后的类的总的内存大小，每个类的内存大小都是计算好的，我们在调用alloc的时候就直接根据这个内存大小直接开辟固定的内存（当然，这个是最简单的叙述，还有其他的需要计算内存的情况），而这个内存的最小值是16个字节，这里苹果是出于什么考虑呢？还有待探究，有谁知道可以教教我哈～～

那么回到我们的例子本身

对于demo1，我们因为赋予了name1值，而他的父类里还有五个变量，按添加ivar的顺序他们在内存中的存储及顺序主要如下：

```
(lldb) x/x demo1 -c 10 -s 8
0x600000a61440: 0x000000010b1493c8(class) 0x0000000000000000(name1)
0x600000a61450: 0x0000000000000000(name2) 0x0000000000000000(nme3)
0x600000a61460: 0x0000000000000000(name4) 0x0000000000000000(name5)
0x600000a61470: 0x000000010b13f698(name) 0x0000000000000000
0x600000a61480: 0x0000bc5812bd1480 0xffffffffffff0057
```

这里可以看到他们每个变量的固定偏移都是八个字节，因为我这里用的__x86_64__，那么实际上我们对象占有的内存大小就是 7 * 8 = 56个字节；当然这个是走了正常alloc的对象，所以内存排布上是正常的。而且因为我们父类的所有属性都没有赋值，所以都是 0，上边也可以验证这个问题。

但是obj这个对象又为什么能够打印呢？且`testPrint`能够正常打印，而`print`却不能打印呢？

```
id cls = [ReplaceSubDemo class];
void *obj = &cls;
[(__bridge id)obj testPrint];
[(__bridge id)obj print];
```

首先解释下为什么能够调用方法：其原因就是我们的obj指针指向的是class的实例，也就是他的首地址就是我们的class，所以调用他的方法其实和普通的调用对象的方法是一样一样的，不会有任何区别，但是能编译主要是因为编译器推断出了这个类型，是编译器的功劳，不是`id`这个关键字的功劳哈

那又为什么`testPrint`能调用，而`print`会crash呢？其实这个我猜测主要是因为iOS内部为了方便内存管理，做了一些字节对齐的操作，在代码里主要表现就是一个类的最小单位是16字节，而我们首地址是我们的class，按照属性的偏移计算，`testPrint`里面访问的是name1，是第一个属性，所以其地址是首地址之后的八个字节，因为我们这个对象不是通过alloc做的创建，所以就只有从首地址开始的16个字节是我们的对象本身，超出部分已经不是这个对象了，所以在访问name1的时候是可以访问的，而访问name的时候就是野指针了。

**Q：那为什么testPrint打印的结果是<ReplaceSubDemo: 0x600000a66940>呢？**

其实这个主要是因为 name1 的首地址就是0x600000a66940，但是这个时候这里并不是string类型，所以我们调nslog的时候是调了这个地址的对象的describetion方法，就把他的类型和地址打印出来了

### 总结

上边这些还需要慢慢看，最好实践下才会明白，我这里简单的总结下内容吧

1.每个类型所需要的内存大小是固定的，内存大小有一个专门的属性来存储，通过正常的alloc或者runtime的alloc等方法创建的对象的内存是正常的。

2.同一个对象的属性的寻址方法是按偏移走的，所以需要注意runtime里的ivar的结构体里面并没有value的值，记录的是对于对象首地址的偏移量，之后会通过寻址的方式来找到对象的值。

3.同一类型的首地址所存储的值一定是一样的，这个地址是发生在链接期间的，不会根据app的运行而改变。

4.一个类的class方法实际上就是把对象的首地址的值与一个MASK做了一次按位与操作，所有该对象的实例都共用这个class的实例。

5.一个类的class实例的前八个字节存储的是元类所在的位置，9-16字节存储的是他的superclass

6.如果对象不能正常的开辟内存，就会造成野指针的问题，所以我们一定要用好alloc new。