---
layout:     post
title:      Objective-C的消息机制
date:       2019-01-12
---

# objc_msgSend执行流程

`OC`中的方法调用，其实都是转换为`objc_msgSend`函数的调用

```objective-c
HPerson *person = [[HPerson alloc] init];
[person personTest];
// objc_msgSend(person, sel_registerName("personTest"))
// @selector(personTest) = sel_registerName("personTest")

[HPerson initialize];
// objc_msgSend(objc_getClass("HPerson"), sel_registerName("initialize"))
// objc_msgSend([HPerson class]), @selector(initialize))
```

`OC`的方法调用：消息机制，给方法调用者发送消息

- 消息接收者（receiver）：person 、[HPerson class] 
- 消息名称：personTest、initialize

`objc_msgSend`的执行流程可以分为3大阶段：

- 消息发送
- 动态方法解析
- 消息转发

`objc_msgSend`如果3个阶段都找不到合适的方法进行调用，会报错`unrecognized selector send to instance`

# 消息发送

`objc_msgSend`源码伪代码实现：

```c
void objc_msgSend(id receiver, SEL selecotr) {
    if (receiver == nil) return;
    
    // 查找缓存
    
    // 缓存查找不到，去class_rw_t遍历methods（方法列表）
}
```

流程图如下：

![avatars](https://ws3.sinaimg.cn/large/006tNc79ly1fz76fsejgmj319m0lmwi8.jpg)

# 动态方法解析

消息发送阶段找不到方法实现，会进入动态方法解析

```objective-c
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (self == @selector(test)) {
        // 获取其他方法
        Method method = class_getInstanceMethod(self, @selector(other));
        // 动态添加test方法的实现
        class_addMethod(self, sel, 
                        method_getImplementation(method),
                        method_getTypeEncoding(method));
        // 也可以调用C语言方法 class_addMethod(self, sel, (IMP)c_other, "v16@0:8");
        // 返回YES代表又动态添加方法
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}

- (void)other {

}

void c_other(id self, SEL _cmd) {
    
}

+ (BOOL)resolveClassMethod:(SEL)sel {
    if (self == @selector(test)) {
        // 第一个参数是object_getClass(self)
        class_addMethod(object_getClass(self), sel, (IMP)c_other, "v16@0:8");
    	return YES;
    }
	return [super resolveInstanceMethod:sel];
}
```

流程图如下：

![avatars](https://ws4.sinaimg.cn/large/006tNc79ly1fz7lvvwc7yj318c0h8myx.jpg)

# 消息转发

消息转发：将消息转发给别人

首先会执行`forwardingTargetForSelector:`方法：

```objective-c
- (id)forwardingTargetForSelector:(SEL)aSelector { // 实例方法调用-号方法，类方法调用+号方法
	// 返回能够处理这个方法的对象
    if (aSelector == @selector(test)) {
        // objc_msgSend(cat, aSelector)
        return [[HCat alloc] init];
    }
    return [super forwardingTargetForSelector:aSelector];

}
```

如果`forwardingTargetForSelector:`返回为`nil`，会执行`methodSignatureForSelector:(SEL)aSelector`方法，方法签名有值的话，就会封装进`NSInvocation`，即会执行`forwardInvocation:`方法，进入`forwardInvocation:`方法之后，可以任意执行任何代码，便不会crash

```objective-c
// 方法签名：返回值类型、参数类型
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    if (aSelector == @selector(test)) {
       return [NSMethodSignature signatureWithObjCTypes:"v16@0:"];
    }
    return [super methodSignatureForSelector:aSelector];
}

// NSInvocation封装了一个方法调用，包括：方法调用者、方法名、方法参数
// anInvocation.target 方法调用者
// anInvocation.selector 方法名
// [anInvocation getArgument:NULL atIndex:0]
- (void)forwardInvocation:(NSInvocation *)anInvocation {
	//anInvocation.target = [[HCat alloc] init];
    //[anInvocation invoke];
    [anInvocation invokeWithTarget:[[HCat alloc] init]];
}
```

流程图如下：

![avatars](https://ws1.sinaimg.cn/large/006tNc79ly1fz7nldxdp8j31aa0kujt5.jpg)

# @dynamic

属性默认会实现`@synthesize age = _age, height = _height;`，即默认生成成员变量

`@dynamic age`提醒编译器不要自动生成`setter、getter`的实现、不要自动生成成员变量

那么就需要在动态解析、消息转发处实现：

```objective-c
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(setAge:)) {
        
    } else if (sel == @selector(age)) {
        
    }
    return [super resolveInstanceMethod:sel];
}
```

# super

```objective-c
- (instancetype)init {
    if (self = [super init]) {
        NSLog(@"[self class] = %@", [self class]); // HStudent
        NSLog(@"[self superclass] = %@", [self superclass]); // HPerson
        
        NSLog(@"[super class] = %@", [super class]); // HStudent
        NSLog(@"[super superclass] = %@", [super superclass]); // HPerson
    }
    return self;
}
```

`class`方法为`NSObject`的方法，最后都会走到`NSObject`中`class`的实现：即`[self class][super class]`最后调用的都为`NSObject`中的`class`

```objective-c
- (Clcass)class {
	return object_getClass(self); // class 底层实现
}
```

返回值取决于`self`，即方法调用者，即`receiver`

```objective-c
[self class]   // objc_msgSend(self, @selector(class))
[super class]  // objc_msgSendSuper({self, [HPerson class]}, @selector(class))
```

`[super class]`的调用，`receiver`依旧为`self`，只不过会直接去父类的类对象里查找`class`方法

**[super message]的底层实现**结论：

1. 消息接收者仍然是子类对象
2. 从父类开始查找方法的实现

`superClass`底层实现为：

```objective-c
- (Class)superclass {
    return calss_getSuperclass(object_getClass(self));
}
```

![avatars](https://ws2.sinaimg.cn/large/006tNc79ly1fz84u8ixyrj30s00tawjx.jpg)

