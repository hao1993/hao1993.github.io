---
layout:     post
title:      Objective-C-Runtime-API
date:       2019-01-18
---

# 类

- 获取isa指向的Class   `Class object_getClass(id obj) `		

- 设置isa指向的Class  `Class object_setClass(id obj, Class cls)`

  ```objective-c
  HPerson *person = [[HPerson alloc] init];
  object_setClass(person, [HPerson class]);
  // 修改person为HCar类型
  ```

- 判断一个OC对象是否为Class `BOOL object_isClass(id obj)`

```objective-c
HPerson *person = [[HPerson alloc] init];    
NSLog(@"%d %d %d",
      object_isClass(person),
      object_isClass([HPerson class]),
      object_isClass(object_getClass([HPerson class]))
      );   // 0  1    1
```

- 判断一个Class是否为元类  `BOOL class_isMetaClass(Class cls)`

- 获取父类 `class_getSuperclass(Class cls)`

- 动态创建一个类（参数：父类，类名，额外的内存空间）

   `Class objc_allocateClassPair(Class superclass, const char *name, size_t extraBytes)`

- 注册一个类（要在类注册之前添加成员变量） `void objc_registerClassPair(Class cls)`
- 销毁一个类   `void objc_disposeClassPair(Class cls)`

```objective-c
	// 创建类
    Class newClass = objc_allocateClassPair([NSObject class], "HDog", 0);
	class_addIvar(newClass, "_age", 4, 1, @encode(int));
	class_addMethod(newClass, @selector(run), (IMP)run, "v@:");
    // 注册类
    objc_registerClassPair(newClass);
	id dog = [[newClass alloc] init];
	// 即动态生成 HDog对象
	NSLog(@"%zd", class_getInstanceSize(newClass));
	[dog setValue:@10 forKey:@"_age"];
	[dog run];

	void run(id self, SEL _cmd) {
    	NSLog(@"_____ %@ - %@", self, NSStringFromSelector(_cmd));
	}	

	// 在不需要这个类时释放
    objc_disposeClassPair(newClass);

	// addIvar必须放在 注册类之前，成员变量在class_ro_t中，是只读的。类的成员变量结构在创建之后，是不能	   	通过runtime创建的。

	//NSString *className = [NSString stringWithFormat:@"H%@",@"abc"];
    //Class newClass = objc_allocateClassPair([NSObject class], className.UTF8String, 0);
```

# 成员变量

- 获取一个实例变量信息  `Ivar class_getInstanceVariable(Class cls, const char *name)`
- 获取成员变量的相关信息
  -  `const char *ivar_getName(Ivar v)`
  - `const char *ivar_getTypeEncoding(Ivar v)`

```objective-c
Ivar ageIvar = class_getInstanceVariable([HPerson class], "_age");
NSLog(@"%s %s", ivar_getName(ageIvar), ivar_getTypeEncoding(ageIvar));
// _age i
```

- 设置和获取成员变量的值 
  - `void object_setIvar(id obj, Ivar ivar, id value)`
  - `id object_getIvar(id obj, Ivar ivar)`

```objective-c
Ivar nameIvar = class_getInstanceVariable([HPerson class], "_name");

HPerson *person = [[HPerson alloc] init];
object_setIvar(person, nameIvar, @"123");
object_setIvar(person, ageIvar, (__bridge id)(void *)10);
NSLog(@"%@ %d", person.name, person.age);

```

- 拷贝实例变量列表（最后需要调用free释放） **常用**

  `Ivar *class_copyIvarList(Class cls, unsigned int *outCount)`

```objective-c
	// 成员变量的数量
    unsigned int count;  
    Ivar *ivars = class_copyIvarList([MJPerson class], &count);   // 指针可以当作数组来用
    for (int i = 0; i < count; i++) {
        // 取出i位置的成员变量
        Ivar ivar = ivars[i];
        NSLog(@"%s %s", ivar_getName(ivar), ivar_getTypeEncoding(ivar));
    }
    free(ivars);
	// runtime里边调用copy或者create，一般需要free
```

以`UITextField`为例：

```objective-c
//    unsigned int count;
//    Ivar *ivars = class_copyIvarList([UITextField class], &count);
//    for (int i = 0; i < count; i++) {
//        // 取出i位置的成员变量
//        Ivar ivar = ivars[i];
//        NSLog(@"%s %s", ivar_getName(ivar), ivar_getTypeEncoding(ivar));
//    }
//    free(ivars);
    
    self.textField.placeholder = @"请输入用户名";
    [self.textField setValue:[UIColor redColor] forKeyPath:@"_placeholderLabel.textColor"];
    
//    UILabel *placeholderLabel = [self.textField valueForKeyPath:@"_placeholderLabel"];
//    placeholderLabel.textColor = [UIColor redColor];
    
	// 查看类型
	id placeholderLabel = [self.textField valueForKeyPath:@"_placeholderLabel"];
    NSLog(@"%@ %@", [placeholderLabel class], [placeholderLabel superclass]);

//    NSMutableDictionary *attrs = [NSMutableDictionary dictionary];
//    attrs[NSForegroundColorAttributeName] = [UIColor redColor];
//    self.textField.attributedPlaceholder = [[NSMutableAttributedString alloc] initWithString:@"请输入用户名" attributes:attrs];
```

实现字典转模型：

```objective-c
+ (instancetype)h_objectWithJson:(NSDictionary *)json
{
    id obj = [[self alloc] init];
    
    unsigned int count;
    Ivar *ivars = class_copyIvarList(self, &count);
    for (int i = 0; i < count; i++) {
        // 取出i位置的成员变量
        Ivar ivar = ivars[i];
        NSMutableString *name = [NSMutableString stringWithUTF8String:ivar_getName(ivar)];
        [name deleteCharactersInRange:NSMakeRange(0, 1)];
        
        // 设值
        id value = json[name];
        if ([name isEqualToString:@"ID"]) {
            value = json[@"id"];
        }
        [obj setValue:value forKey:name];
    }
    free(ivars);
    
    return obj;
}
```

# 属性

- 获取一个属性 `objc_property_t class_getProperty(Class cls, const char *name)`

- 拷贝属性列表（最后需要调用free释放）

  `objc_property_t *class_copyPropertyList(Class cls, unsigned int *outCount)`

- 动态添加属性

  `BOOL class_addProperty(Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount)`

- 动态替换属性

  `void class_replaceProperty(Class cls, const char *name, const objc_property_attribute_t *attributes,unsigned int attributeCount)`

- 获取属性的一些信息
  - `const char *property_getName(objc_property_t property)`
  - `const char *property_getAttributes(objc_property_t property)`

# 方法

- 动态添加方法  `BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)`
- 动态替换方法 `IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types)`

```objective-c
class_replaceMethod([HPerson class], @selector(run), (IMP)myrun, "v");
```

- 拷贝方法列表（最后需要调用free释放）

  `Method *class_copyMethodList(Class cls, unsigned int *outCount)`

- 获取方法的相关信息（带有copy的需要调用free去释放）
  - `SEL method_getName(Method m)`
  - `IMP method_getImplementation(Method m)`
  - `const char *method_getTypeEncoding(Method m)`
  - `unsigned int method_getNumberOfArguments(Method m)`
  - `char *method_copyReturnType(Method m)`
  - `char *method_copyArgumentType(Method m, unsigned int index)`

- 获得一个实例方法、类方法
  - `Method class_getInstanceMethod(Class cls, SEL name)`
  - `Method class_getClassMethod(Class cls, SEL name)`
- 用block作为方法实现
  - `IMP imp_implementationWithBlock(id block)`
  - `id imp_getBlock(IMP anImp)`
  - `BOOL imp_removeBlock(IMP anImp)`

```objective-c
    HPerson *person = [[HPerson alloc] init];
    class_replaceMethod([HPerson class], @selector(run), imp_implementationWithBlock(^{
        NSLog(@"123123");
    }), "v");
    [person run];
```

- 选择器相关
  - `const char *sel_getName(SEL sel)`
  - `SEL sel_registerName(const char *str)`

- 方法实现相关操作
  - `IMP class_getMethodImplementation(Class cls, SEL name)`
  - `IMP method_setImplementation(Method m, IMP imp)`
  - `void method_exchangeImplementations(Method m1, Method m2) `

```objective-c
HPerson *person = [[HPerson alloc] init];
        
Method runMethod = class_getInstanceMethod([HPerson class], @selector(run));
Method testMethod = class_getInstanceMethod([HPerson class], @selector(test));
method_exchangeImplementations(runMethod, testMethod);

[person run];
```

`method_exchangeImplementations`交换的是Class_rw_t里面的methods数组里边的method_t里的IMP imp(指向函数的指针，函数地址)

调用此方法之后，会清空缓存

源码如下：

![avatars](https://ws4.sinaimg.cn/large/006tNc79ly1fzbq8dvzr1j30xo0i6jwy.jpg)