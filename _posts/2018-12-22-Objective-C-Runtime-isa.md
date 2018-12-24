---
layout:     post
title:      Objective-C Runtime-isa详解
date:       2018-12-22
---

# Runtime简介

- Objective-C是一门动态性比较强的编程语言，跟C、C++等语言有着很大的不同
- Objective-C的动态性是由Runtime API来支撑的
- Runtime API提供的接口基本都是C语言的，源码由C\C++\汇编语言编写

Runtime[源码下载](https://opensource.apple.com/tarballs/objc4/)。

# isa简介

要想学习Runtime，首先要了解它底层的一些常用数据结构，比如isa指针。

在arm64架构之前，isa就是一个普通的指针，存储着Classs、Meta-Class对象的内存地址。

从arm64架构开始，对isa指针进行了优化，变成了一个共用体（union）结构，还使用位域来存储更多的信息。

打开Runtime源码，全局搜索`objc_object {`，在`objc-private.h`中找到

```objective-c
struct objc_object {
private:
    isa_t isa;
}
```

`isa_t`点进去可查看isa结构如下：

```objective-c
union isa_t 
{
    Class cls;
    uintptr_t bits;
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19;
    };
};
```

接下来看一下使用共用体的目的及好处

# 需求分析

一般情况下一个类属性的存取如下：

```objective-c
@interface TestPerson : NSObject
@property (assign, nonatomic, getter=isTall) BOOL tall;
@property (assign, nonatomic, getter=isRich) BOOL rich;
@property (assign, nonatomic, getter=isHansome) BOOL handsome;
@end
   
@implementation TestPerson    
@end
    
@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
	
    TestPerson *person = [[TestPerson alloc] init];
    person.rich = YES;
    person.tall = NO;
    person.handsome = NO;
    NSLog(@"tall:%d rich:%d hansome:%d", person.isTall, person.isRich,                			person.isHandsome);
}
@end
    
```

此时打印一下`person`对象占用的内存空间，导入头文件`#import <objc/runtime.h>`

```objective-c
NSLog(@"%zd", class_getInstanceSize([TestPerson class]));
```

结果为**16**，其中`isa`指针占8个字节，一个`BOOL`类型占1个字节，8 + 3 = 11，由于内存对齐原则，故为16。

一个`BOOL`类型的结果，为0或1，既然如此，我们就可以用一个二进制位来存储一个`BOOL`类型。这样三个属性用3个二进制位来存储，3个合起来，只需要用一个字节就可以了。

一种比较容易理解的方式可以这么来实现：

定义一个char类型的成员变量_tallRichHansome，二进制为0b0000 0011，从右至左，最右边一位代表tall，接着代表rich，hansome，即`person.tall = YES; person.rich = YES; person.handsome = NO;`

```objective-c
@interface TestPerson : NSObject {
    char _tallRichHansome;
}
- (void)setTall:(BOOL)tall;
- (void)setRich:(BOOL)rich;
- (void)setHandsome:(BOOL)handsome;

- (BOOL)isTall;
- (BOOL)isRich;
- (BOOL)isHandsome;
@end

@implementation TestPerson  
- (void)setTall:(BOOL)tall {
    
}

- (BOOL)isTall {
    
}

- (void)setRich:(BOOL)rich {
    
}

- (BOOL)isRich {
    
}

- (void)setHandsome:(BOOL)handsome {
    
}

- (BOOL)isHandsome {
    
}
@end
    
```

# 取值操作

我们可以利用位运算来进行取值操作，同为1，结果才为1，否则为0。

```c
 0011
&1111
-----
 0011    
```

位运算可以取出特定位的值。0b0000 0011，如果要取出tall的值，那么可以用0b0000 0001

```c
 0b0000 0011
&0b0000 0001
------------
 0b0000 0001    
```

即：

```objective-c
- (BOOL)isTall {
    return _tallRichHansome & 1;
}

- (BOOL)isRich {  // 0b0000 0010为2  
    // _tallRichHansome & 2 要么为0，要么为2即有值
   //return (BOOL)_tallRichHansome & 2;   可强制转换，也可取两次!操作
    return !!(_tallRichHansome & 2);
}

- (BOOL)isHandsome {  // 0b0000 0100为4 
    return !!(_tallRichHansome & 4);
}
```

验证一下这种做法是否正确：

```objective-c
@implementation TestPerson 
- (instancetype)init
{
    if (self = [super init]) {
        _tallRichHansome = 0b00000011;
    }
    return self;
}
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
	
    TestPerson *person = [[TestPerson alloc] init];
    NSLog(@"tall:%d rich:%d hansome:%d", person.isTall, person.isRich,                			person.isHandsome);
}
@end 
```

输出结果为：

```objective-c
tall:1 rich:1 hansome:0
```

`set`方法中，1，2，4比较抽象，用宏定义对其进行优化：

```objective-c
// mask 掩码，一般用来按位与(&)运算的
#define TTallMask 1
#define TRichMask 2
#define THandsomeMask 4
```

1，2，4依旧不容易搞懂代表哪一位，那么可以这么做：

```objective-c
#define TTallMask     0b00000001
#define TRichMask     0b00000010
#define THandsomeMask 0b00000100
```

这样还是比较抽象，一般来说这么做：

```objective-c
// 1<<0 代表1左移0位，1<<1 代表1左移1位，0b00000001 左移1位为 0b00000010
#define TTallMask     1<<0
#define TRichMask     1<<1
#define THandsomeMask 1<<2
```

`1<<0`最好用小括号括起来，比如 `!!_tallRichHansome & 1 << 0`，编译器运算结果会异常。以下为最终版：

```objective-c
#define TTallMask     (1<<0)
#define TRichMask     (1<<1)
#define THandsomeMask (1<<2)
```

`set`方法修改为：

```objective-c
- (BOOL)isTall {
    return !!(_tallRichHansome & TTallMask);
}

- (BOOL)isRich {
    return !!(_tallRichHansome & TRichMask);
}

- (BOOL)isHandsome {  
    return !!(_tallRichHansome & THandsomeMask);
}
```

接着再运行一下，看打印结果，依然为：

```objective-c
tall:1 rich:1 hansome:0
```

# 存值操作

```objective-c
TestPerson *person = [[TestPerson alloc] init];
person.rich = YES;
person.tall = YES;
person.handsome = YES;
NSLog(@"tall:%d rich:%d hansome:%d", person.isTall, person.isRich,                			person.isHandsome);
```

外边都为YES，如何把值存进去呢。

rich为YES，即0b00000001，如何把1存进去呢。这里用到`|`运算。

```c
 0b 0000 0001
|0b 0000 0001
--------------
 0b 0000 0001    
```

想将某一位置为1的话，把掩码的**相应位**置为1即可。

```
- (void)setTall:(BOOL)tall {
    if (tall) {
		_tallRichHansome |= TTallMask;//_tallRichHansome = _tallRichHansome | TTallMask;
	}
}
```

rich为NO，即0b00000000，如何把0存进去。

```c
 0b 0000 0000
&0b 1111 1110
--------------
 0b 1111 1110    
```

 即拿到掩码取反，然后进行按位与操作，取反字符为`~`

```
- (void)setTall:(BOOL)tall {
    if (tall) {
		_tallRichHansome |= TTallMask;
	} else {
        _tallRichHansome &= ~TTallMask;//_tallRichHansome=_tallRichHansome & ~TTallMask; 
	}
}

- (void)setRich:(BOOL)rich {
    if (rich) {
		_tallRichHansome |= TRichMask;
	} else {
        _tallRichHansome &= ~TRichMask; 
	}
}

- (void)setHandsome:(BOOL)handsome {
    if (handsome) {
		_tallRichHansome |= THandsomeMask;
	} else {
        _tallRichHansome &= ~THandsomeMask;
	}
}
```

运行：

```objective-c
TestPerson *person = [[TestPerson alloc] init];
person.rich = YES;
person.tall = YES;
person.handsome = YES;
NSLog(@"tall:%d rich:%d hansome:%d", person.isTall, person.isRich,                			person.isHandsome);
```

结果为：：

```objective-c
tall:1 rich:1 hansome:1
```

# 位域

如果新增一个变量的话，就需要实现set、get方法，同时增加掩码，这样的话也比较简单，但是可读性比较差。这样来实现，可读性会好很多。

```objective-c
@interface TestPerson () {
    struct {
        char tall;
        char rich;
        char hansome;
    } _tallRichHansome;
}
@end
```

这样的话就占了三个字节，跟之前set、get方法一样，结构体是支持位域技术的，如此便只占一个字节：

```objective-c
@interface TestPerson () {
    struct {
        char tall : 1;   // 代表只占一位
        char rich : 1;
        char handsome : 1;
    } _tallRichHandsome;  // 0b0000 0000 tall会自动排在最右边，从右至左为rich，handsome
}
@end
```

set、get方法可修改为：

```objective-c
- (void)setTall:(BOOL)tall {
    _tallRichHandsome.tall = tall;
}

- (BOOL)isTall {
    return _tallRichHandsome.tall;
}

- (void)setRich:(BOOL)rich {
    _tallRichHandsome.rich = rich;
}

- (BOOL)isRich {
    return _tallRichHandsome.rich;
}

- (void)setHandsome:(BOOL)handsome {
    _tallRichHandsome.handsome = handsome;
}

- (BOOL)isHandsome {
    return _tallRichHandsome.handsome;
}
```

运行：

```objective-c
TestPerson *person = [[TestPerson alloc] init];
person.rich = YES;
person.tall = NO;
person.handsome = NO;
NSLog(@"tall:%d rich:%d hansome:%d", person.isTall, person.isRich,                			person.isHandsome);
```

结果为：

```objective-c
tall:0 rich:-1 hansome:0
```

tall与hansome值是正常的，rich为-1，为什么呢？

打断点可知，值设置成功了

![isa00](https://ws2.sinaimg.cn/large/006tNbRwly1fyh1dgg5m9j312u0fcta6.jpg)

查看内存地址：

![isa01](https://ws3.sinaimg.cn/large/006tNbRwly1fyh1ep4gslj30zk07ajs3.jpg)

查看计算器：

![isa02](https://ws1.sinaimg.cn/large/006tNbRwly1fyh1f4mz4tj30l209amzy.jpg)

刚刚是给rich赋值为1，即0b0000 0010

rich为-1的原因：

```objective-c
- (BOOL)isRich {
    // bool类型为一个字符，8个字节 0b0000 0000 ,
    // _tallRichHandsome.rich只有一个二进制位  _tallRichHandsome.rich = 0b1 ，即0b0000 0001
    // 1会被当作符号为去覆盖，结果为0b1111 1111，为-1
    return _tallRichHandsome.rich;
}
```

可修改为之前的方式，添加两个!!

```objective-c
- (BOOL)isRich {
    return !!_tallRichHandsome.rich;
}
```

不添加!!也可解决，修改位域：

```objective-c
@interface TestPerson () {
    struct {
        char tall : 2;   
        char rich : 2;
        char handsome : 2;
    } _tallRichHandsome; 
}
@end
```

运行：

![isa03](https://ws1.sinaimg.cn/large/006tNbRwly1fyh1fhwevtj312e0i8abu.jpg)

![isa04](https://ws3.sinaimg.cn/large/006tNbRwly1fyh1fs9jvmj30la096juc.jpg)

位域为何修改为两位就可以了呢？

```objc
- (BOOL)isRich {
    // _tallRichHandsome.rich只有一个二进制位  _tallRichHandsome.rich = 0b01 ，即0b0000 0001
    // 0会被当作符号为去覆盖，结果为0b0000 0001，为1
    return _tallRichHandsome.rich;
}
```

# 共用体

**共用体**：在进行某些算法的C语言编程的时候，需要使几种不同类型的变量存放到同一段内存单元中。也就是使用覆盖技术，几个变量互相覆盖。这种几个不同的变量共同占用一段内存的结构，在C语言中，被称作“共用体”类型结构，简称共用体。

苹果官方使用的是共用体：

```objc
#define TTallMask (1<<0)
#define TRichMask (1<<1)
#define THandsomeMask (1<<2)

@interface MJPerson() {
    union {
        char bits;
        struct {
            char tall : 1;
            char rich : 1;
            char handsome : 1;
        };
    } _tallRichHandsome;
}
@end

@implementation MJPerson

- (void)setTall:(BOOL)tall {
    if (tall) {
        _tallRichHandsome.bits |= TTallMask;
    } else {
        _tallRichHandsome.bits &= ~TTallMask;
    }
}

- (BOOL)isTall {
    return !!(_tallRichHandsome.bits & TTallMask);
}

- (void)setRich:(BOOL)rich {
    if (rich) {
        _tallRichHandsome.bits |= TRichMask;
    } else {
        _tallRichHandsome.bits &= ~TRichMask;
    }
}

- (BOOL)isRich {
    return !!(_tallRichHandsome.bits & TRichMask);
}

- (void)setHandsome:(BOOL)handsome {
    if (handsome) {
        _tallRichHandsome.bits |= THandsomeMask;
    } else {
        _tallRichHandsome.bits &= ~THandsomeMask;
    }
}

- (BOOL)isHandsome {
    return !!(_tallRichHandsome.bits & THandsomeMask);
}

@end
```

运行程序验证结果可知，这样也是可以的。

使用共用体的好处：

- 运算的时候是直接通过位运算来做，这样就非常精准，不会出现不准确的情况；
- 里边的结构体增加可读性；

利用位域增加可读性，利用位运算存取值比较直接。

# 总结

回过头来看isa的结构：

```objective-c
union isa_t 
{
    Class cls;
    uintptr_t bits;
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19;
    };
};
```

`bits`用来存放所有的数据。

- **nonpointer**：0，代表普通的指针，存储着Class、Meta-Class对象的内存地址
  ​			 1，代表优化过，使用位域存储更多的信息

- **has_assoc**：是否有设置过关联对象，如果没有，释放时会更快
- **has_cxx_dtor**：是否有C++的析构函数（.cxx_destruct），如果没有，释放时会更快
- **shiftcls**：存储着Class、Meta-Class对象的内存地址信息
- **magic**：用于在调试时分辨对象是否未完成初始化
- **weakly_referenced**：是否有被弱引用指向过，如果没有，释放时会更快
- **deallocating**：对象是否正在释放
- **has_sidetable_rc**：引用计数器是否过大无法存储在isa中，如果为1，那么引用计数会存储在一个叫SideTable的类的属性中
- **extra_rc**：里面存储的值是引用计数器减1

总结：

- 在arm64之前，isa只是个指针，里边存储着类对象、元类对象的地址值；

- arm64开始，isa经过了优化，它采取共用体的结构，将一个64位的内存数据分开来存储了很多东西，其中			   的33位是拿来存储类对象、元类对象的地址值。