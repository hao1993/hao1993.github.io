---
layout:     post
title:      Objective-C-Runtime-Class的结构
date:       2019-01-06
---

# Class的结构

类对象的本质是一个结构体：

```objective-c
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // 方法缓存
    class_data_bits_t bits;    // 用于获取具体的类信息
}
```

`bits` & `FAST_DATA_MASK`可以取出`class_rw_t`：

```objective-c
class_rw_t* data() {
    return (class_rw_t *)(bits & FAST_DATA_MASK);
}
```

`class_rw_t`的结构如下：

```objective-c
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;      //方法列表
    property_array_t properties; //属性列表 
    protocol_array_t protocols;  //协议列表

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;
}
```

`class_ro_t`的结构如下：

```objective-c
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;            // instance对象占用的内存空间
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;					// 类名
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;			// 成员变量列表

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};
```

# 详解

## class_rw_t

`class_rw_t`里面的methods、properies、protocols是二维数组，是可读可写的，包含了类的初始内容、分类的内容

![avatar](https://ws3.sinaimg.cn/large/006tNc79ly1fz6g0t1tg5j313g0b0dh2.jpg)

## class_ro_t

`class_ro_t`里面的baseMethodList、baseProtocols、ivars、baseProperties是一维数组，是只读的，包含了类的初始内容

![avatars](https://ws4.sinaimg.cn/large/006tNc79ly1fz6g5d4cboj30xa0740ue.jpg)

一开始编译的时候，是有ro_t的，之后ro_t会被赋值为rw_t二维数组中的一个。

## method_t

`method_t`是对方法\函数的封装

```objective-c
struct method_t {
    SEL name;    			// 函数名
    const char *types;		// 编码（返回值类型、参数类型）
    IMP imp;				// 指向函数的指针（函数地址）
};
```

- `IMP`代表函数的具体实现：

  ```objective-c
  typedef id _Nullable (*IMP)(id _Nonnull, SEL _NOnnull,...);
  ```

- `SEL`代表方法\函数名，一般叫做选择器，地层结构跟char *类似：

  - 可以通过`@selector()`和`sel_registerName()`获得；

    ```objective-c
    SEL sel1 = sel_registerName("test");
    SEL sel2 = @selector(test);
    ```

  - 可以通过`sel_getName()`和`NSStringFromSelector()`转成字符串；
  - 不同类中相同名字的方法，所对应的方法选择器是相同的。

- `types`包含了函数返回值、参数编码的字符串

  返回值 参数1 参数2 … 参数n

```objective-c
// v 16 @ 0 : 8
// void id SEL
[person test];
```

iOS中提供了一个叫做@encode的指令，可以将具体的类型表示成字符串编码

## 方法缓存

`Class`内部结构中有个方法缓存（cache_t），用**散列表（哈希表）**来缓存曾经调用过的方法，可以提高方法的查找速度

```objective-c
struct cache_t {
    struct bucket_t *_buckets; // 散列表
    mask_t _mask;			   // 散列表的长度 -1
    mask_t _occupied;		   // 已经缓存的方法数量
};

struct bucket_t {
    cache_key_t _key;		// SEL作为key
    IMP _imp;				// 函数的内存地址
};
```

牺牲内存空间来换取时间

调用的是哪个就缓存哪个