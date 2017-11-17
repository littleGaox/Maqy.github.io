---
layout:     post
title:      transform属性详解
subtitle:   
date:       2017-11-17
author:     Maqy
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - iOS
    - transform
---

### transform使用方法

```
@interface UIView : UIResponder
@property(nonatomic) CGAffineTransform transform;   // default is CGAffineTransformIdentity. animatable
@end
```
说明：`transform`属性主要用于形变，位移和旋转，可用于动画展示，主要方法有
```
    [UIView animateWithDuration:2 animations:^{
        /*
         以下三点结论基于未旋转的情况：
         1.当参数x>0 && x<=M_PI时,为顺时针
         2.当参数x>-M_PI && x<0时,为逆时针
         3.若参数x<M_PI || x>2.0*M_PI时,则旋转方向等同于x%2的旋转方向
         总结：旋转方向就是向最短路径方向旋转
         */
        but.transform = CGAffineTransformMakeRotation(M_PI);// 顺时针旋转180度
    }];
    [UIView animateWithDuration:2 animations:^{
        but.transform = CGAffineTransformMakeScale(2, 1);//宽高伸缩比例
    }];
    [UIView animateWithDuration:2 animations:^{
        but.transform = CGAffineTransformMakeTranslation(4, 6);//xy移动距离
    }];
    [UIView animateWithDuration:2 animations:^{
        but.transform = CGAffineTransformMake(1, 1, 1, 1, 1, 1);//自定义形变,参数自拟，下边会详细描述
    }];
```
一般情况下的动画就是基于以上方法实现

---
### position&anchorPoint

动画旋转操作时是以锚点为中心的，下边详细介绍：
系统表现动画时都是用layer来表现的，包括绘图等也是基于layer的，而UIView可以说是layer的一个容器
而layer有两个比较重要的属性

```
/* 
 The position in the superlayer that the anchor point of the layers
 bounds rect is aligned to. Defaults to the zero point. Animatable.*/
@property CGPoint position;//当前layer的锚点在superlayer伤的位置

/* Defines the anchor point of the layer's bounds rect, as a point in
 * normalized layer coordinates - '(0, 0)' is the bottom left corner of
 * the bounds rect, '(1, 1)' is the top right corner. Defaults to
 * '(0.5, 0.5)', i.e. the center of the bounds rect. Animatable. */
@property CGPoint anchorPoint;//这个就是锚点，只不过其属性值代表其在自己的layer中的比例，比如若是左上角就是(0,0)，默认为(0.5,0.5)，不能超过1，不能小于0
```
而动画是基于锚点来进行形变的，所以只能改变锚点的值，比如绕左上角旋转则如下
```
    but.layer.position = but.frame.origin;
    but.layer.anchorPoint = CGPointMake(0, 0);
    [UIView animateWithDuration:2 animations:^{
        but.transform = CGAffineTransformMakeRotation(M_PI);// 顺时针旋转180度
    }];
```
为什么要改变`position`的值？
因为`position`和`anchorPoint`是两个不同属性，且当其中一个改变时，另一个并不会变，例如上边的`but`，若其`frame=CGRectMake(50,50,100,100)`，则默认`anchorPoint=CGPointMake(0.5,0.5);position=CGPointMake(100,100);`
若此时只改变`anchorPoint=CGPointMake(0,0);`则其`position`不变，那么就会发生位移，位移后`frame=CGRectMake(100,100,100,100);`
所以，若不想发生位移，就需要预先改变`position`和`anchorPoint`。

---
### 旋转后frame值

视图旋转后，其`frame`可能会变大，大小为其外接矩形大小，因为外接矩形肯定是水平的，而旋转后的视图可能并不是水平的了

---
### transform是如何工作的？

在解答这个问题之前，需要补充一个概念
齐次矩阵：所谓齐次坐标就是将一个原本是n维的[向量]用一个n+1维向量来表示。许多图形应用涉及到[几何变换]，主要包括平移、旋转、缩放。那么形变时就可以转换为矩阵相加相乘来做，这样可以减少运算量，提高效率。
看苹果文档：

```
 struct CGAffineTransform { 
    CGFloat a;
    CGFloat b; 
    CGFloat c; 
    CGFloat d; 
    CGFloat tx; 
    CGFloat ty;
 };
 typedef struct CGAffineTransform CGAffineTransform;
```
transform主要是一个结构体，有六个变量，对应一个3*3矩阵：

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1276523-4ff995fb9e6f16b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
而将layer上的坐标［x,y］看成一个三维坐标[x,y,1],并将每个坐标点与上边的transform构成的矩阵相乘就构成了形变后的图形。

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1276523-85e8d748c979202e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
转换前与转换后的z坐标都是1，因为z=x*0+y*0+1*1;所以'transform'构成的矩阵第三列要填001
矩阵相乘后：
x' = ax + cy + tx;  y' = bx + dy + ty;
所以可以推测：
`CGAffineTransformMakeTranslation(4, 6);`构造的位移矩阵中
```
x' = x + tx; //即a=1,c=0,tx = 4;
y' = y + ty; //即b=1,d=0,ty=6;
```
其他矩阵也可根据公式推测出，旋转的较复杂，有什么cos，sin的，数学不好不记了。
当然，若你想要更丰富的形变，只要构造自己的矩阵就可以了，用数学公式。

---
如果形变后想恢复可使用
`self.transform = CGAffineTransformIdentity;`

---
```
#define M_E         2.71828182845904523536028747135266250   /* e              */
#define M_LOG2E     1.44269504088896340735992468100189214   /* log2(e)        */
#define M_LOG10E    0.434294481903251827651128918916605082  /* log10(e)       */
#define M_LN2       0.693147180559945309417232121458176568  /* loge(2)        */
#define M_LN10      2.30258509299404568401799145468436421   /* loge(10)       */
#define M_PI        3.14159265358979323846264338327950288   /* pi             */
#define M_PI_2      1.57079632679489661923132169163975144   /* pi/2           */
#define M_PI_4      0.785398163397448309615660845819875721  /* pi/4           */
#define M_1_PI      0.318309886183790671537767526745028724  /* 1/pi           */
#define M_2_PI      0.636619772367581343075535053490057448  /* 2/pi           */
#define M_2_SQRTPI  1.12837916709551257389615890312154517   /* 2/sqrt(pi)     */
#define M_SQRT2     1.41421356237309504880168872420969808   /* sqrt(2)        */
#define M_SQRT1_2   0.707106781186547524400844362104849039  /* 1/sqrt(2)      */
```