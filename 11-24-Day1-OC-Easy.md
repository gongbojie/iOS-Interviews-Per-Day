# 问题：`@strongify` 和 `@weakify` 两个宏前面的 `@` 是怎么实现的？

答：定义宏的时候用 `try` 包装了一下而已。

# 延伸

## 1. 引言

常常在某些项目或第三方库代码中见到 `@weakify(self);` 和 `@strongify(self);` 这样的代码，回忆起 Objective-C 的语法，好像没有这样的用法。

## 2. 循环引用

Objective-C 通过引用计数（Reference Count）来管理堆内存。当我们创建对象时，系统会为该对象分配一块内存，Objective-C 会维护该对象的引用计数，当该对象的引用计数大于 0 时，其所在内存不会被系统回收；当引用计数为 0 时，其所在内存会被系统回收，该对象被销毁。

假设一种场景，A对象强引用了B对象，那么即使没有别的对象对B进行强引用，B的引用计数至少为1；如果此时B对象也对A进行了强引用，那么A对象的引用计数也至少为1，A，B的引用计数都将永远不会降低到0，也就是永远不会被释放，如果A、B对象不再被使用，没有任何别的对象引用它们，这将导致A、B所分配的内存泄漏。这就是循环引用导致的内存泄漏。

一种常见的循环引用就是`block`属性：假设一个对象有一个强引用的`block`属性，同时此`block`通过`self`指针访问了该对象的其它属性，这将导致`block`也对该对象进行了强引用。

示例代码：

```objC
@interface SecondController ()

@property (nonatomic, assign) NSTimeInterval time;
// self 对象强引用 block 属性。
@property (nonatomic, copy) void(^block)(void);

@end

@implementation SecondController

- (void)viewDidLoad {
  [super viewDidLoad];

  self.time = [[NSDate date] timeIntervalSince1970];
  self.block = ^(){
    // block 属性强引用 self 对象，构成循环引用
    NSLog(@"%f", self.time);
  };
}

@end

```

本例中的问题将导致SecondViewController对象内存泄漏，即使其被弹出，不再被使用，为其分配的内存仍不会被释放。

## 3. 使用 `_weak` 消除循环引用

那如何解决这个问题。
使用 `_weak` 关键字在 `block` 中不直接使用 `self` 指针，而是使用一个被 `_weak` 关键字修饰的指针，此时 `block` 对 `self` 是弱引用，打破了循环引用的闭环。

修改后的代码：

```objC
@interface SecondController ()

@property (nonatomic, assign) NSTimeInterval time;
// self 对象强引用 block 属性。
@property (nonatomic, copy) void(^block)(void);

@end

@implementation SecondController

- (void)viewDidLoad {
  [super viewDidLoad];

  self.time = [[NSDate date] timeIntervalSince1970];
  __weak __typeof__(self) self_weak_ = self;
  self.block = ^(){
    // block 属性弱引用 self_weak_ 指针指向的对象，不再引起循环引用。
    NSLog(@"%f", self_weak_.time);
  };
}

@end

```

## 使用`__strong`避免`block`代码执行一半，`self`指向的对象被释放

到目前为止，我们使用了关键字 `__weak`，问题好像已经被轻松地解决了。对于一般情况，确实是这样。

考虑如下这样定义`block`代码呢？

```objC
self.block = ^{
  dispatch_after(dispatch_time(DISPATCH_TIME_NOW, NSEC_PER_SEC * 1), dispatch_get_main_queue(), ^{
    NSLog(@"====%f", self_weak_.time);
  });
}
```

在新定义的`block`中，`NSLog`函数并不会在调用`block()`时立即执行，而是延迟一秒之后。一秒后，可能`self`所指向的对象已经被释放，`NSLog`的输入将不能满足我们的需求。

那能不能做到既避免循环引用，而又避免输出异常。

修改代码如下：

```objC
__weak __typeof_(self) self_weak_ = self;
self.block = ^{
  // 关键字 __strong 可忽略掉，因为定义一个指针默认就是 __strong 类型的。
  __strong __typeof__(self) self_strong_ = self_weak_;
  dispatch_after(dispatch_time(DISPATCH_TIME_NOW, NSEC_PER_SEC * 1), dispatch_get_main_queue(), ^{
    NSLog(@"===%f", self_strong_.time)
  });
};
```

可以看到，我们在`bolck`里定义了`self_strong_`指针，它和`self_weak_`相反，能够对指向的对象进行强引用。这将意味着，只要开始执行`block`代码，直到`block`代码执行完毕前，`self`指向的对象都不会被释放，`NSLog`函数能输出正确的结果。

## 5. 宏定义`weakify`和`strongify

`weakify`和`strongify`是两个宏定义，最早出现在库[libextobjc](https://github.com/jspahrsummers/libextobjc)中，后被广泛采用。一般定义如下：

```objC
#ifndef keywordify
#in DEBUG
#define keywordify autoreleasepool {}
#else
#define keywordify try {} @catch (...) {}
#endif
#endif

#ifndef weakify
#if __has_feature(objc_arc)
#define weakify(object) keywordify __weak __typeof__(object) object##_##weak_ = object;
#else
#define weakify(object) keywordify __block __typeof__(object) object##_##block_ = object;
#endif
#endif

#ifndef strongify
#if _has_feature(objc_arc)
#define strongify(object) keywordify __typeof__(object) object = object##_##weak_;
#else
#define strongify(object) keywordify __typeof__(object) object = object##_##block_;
#endif
#endif

```

可以看出，这两个宏定义对于测试环境和正式环境，ARC环境和非ARC环境分别进行了定义，先将它们分类整理如下：

```objC
//1. 测试且 ARC 环境
#ifndef weakify
#define weakify(object) autoreleasepool {} __weak __typeof__(object) object##_##weak_ = object;
#endif

#ifndef strongify
#define strongify(object) autoreleasepool {} __typeof__(object) object = object##_##weak_;
#endif

//2. 正式且 ARC 环境
#ifndef weakify
#define weakify(object) try {} @catch (...){} __weak __typeof__(object) object##_##weak_ = object;
#endif

#ifndef strongify
#define strongify(object) try {} @catch (...){} __typeof__(object) object = object##_##weak_;
#endif

//3. 测试且非 ARC 环境
#ifndef weakify
#define weakify(object) autoreleasepool {} __block __typeof__(object) object##_##block_ = object;
#endif

#ifndef strongify
#define strongify(object) autoreleasepool {} __typeof__(object) object = object##_##block_;
#endif

//4. 正式且非 ARC 环境
#ifndef weakify
#define weakify(object) try {} @catch (...){} __block __typeof__(object) object##_##block_ = object;
#endif

#ifndef strongify
#define strongify(object) try {} @catch (...){} __typeof__(object) object = object##_##block_;
#endif

```

可以看到，在宏定义中，在指针之前，测试环境加入了代码 `autoreleasepool {}`，正式环境加入了代码`try {} @catch (...){}`。加入它们的目的是为了在使用`weakify`或`strongify`时，吃掉前面加上的`@`符号。



在`autoreleasepool {}`前加上`@`，刚好为`@autoreleasepool {}`；在`try {} @catch (...){} `前加上`@`，刚好为`@try {} @catch (...){}`，符合 Objective-C 语法。参见[链接](https://github.com/jspahrsummers/libextobjc/issues/31)。

> Ask:I wanted to know the reason for the `try {}` `finally {}` in each macro. It seems somewhat arbitrary but I am not a macro expert.
>
> Answer:Since the macros are intended to be used with an `@` preceding them (like `@strongify(self);`), the `try {}` soaks up the symbol so it doesn't cause syntax errors.



在 Debug、ARC 环境中，代码`@weakify(self)`等价于 `@autoreleasepool {} __weak __typeof__(self) self_weak_ = self;`。



在 Debug、ARC 环境中，代码`@strongify(self)`等价于 `@autoreleasepool {} __typeof__(self) self = self_weak_;`。

在测试环境使用`@autoreleasepool {}`，而不使用`@try {} @catch (...){}`，因为空的`try catch`代码会产生编译器警告。在正式环境使用`@try {} @catch (...){}`而不使用`@autoreleasepool {}`，是因为`@autoreleasepool {}`创建`autorelease pool`会增加系统开销。

## 6. 使用 weakify 和 strongify

使用示例：

```objC
@weakify(self)
self.block = ^{
  @strong(self)
  dispatch_after(dispatch_time(DISPATCH_TIME_NOW, NSEC_PER_SEC * 1), dispatch_get_main_queue(), ^{
    NSLog(@"====%f", self.time);
  });
};
```

在Debug、ARC环境，代码`@weakify(self)`，在预编译阶段会被替换成代码：

```c
@autoreleasepool {} __weak __typeof__(self) self_weak_ = self; // 其中 @autorelease pool {} 可以忽略
```

在Debug、ARC环境，代码@strongify（self），在预编译阶段会被替换成代码：

```c
@autoreleasepool {} __typeof__(self) self = self_weak_; // 其中 @autoreleasepool {} 可以忽略
```

在`block`外，`@weakify(self)`定义了`__weak`类型的指针`self_weak_`；在`block`内`@strongify(self)`使用`self_weak_`来定义`__strong`类型的指针`self`。值得注意的是，在`block`的作用域内定义的`self`将覆盖作用域更广的类中内置的`self`指针。因此在`@strongify(self)`之后，直接使用`self`指针并不会导致循环引用。

另外，除了可以对`self`指针使用`weakify`和`strongify`，对其他普通指针一样可以使用。如：

```c
UIViewController *vc = [[UIViewController alloc] init];

@weakify(vc)
@strongify(vc)
```

## 7. 总结

`weakify`、`strongify` 为两个宏定义，使用它们是为了解决使用`block`时常遇到的两个问题：

1. 在属性`block`中引用其它兄弟属性导致循环引用的问题（使用`@weakify`中定义的`weak`指针）；

2. 在开始执行 `block` 代码后，`block` 所引用的对象被释放，导致 block 执行结果偏离预期（在 `block`中使用 `@strongify`所定义的`strong`指针）。

在 `weakify`、`strongify` 宏定义中，还使用了 `autoreleasepool {}` 和 `try {} @catch (...){}`，目的是为了在使用 `weakify`、`strongify` 时能够在前面加上 `@` 符号。

其中测试环境使用 `autoreleasepool`，避免空 `try catch` 带来的编译警告；在正式环境使用 `try {} @catch (...){}`，避免创建 `autoreleasepool` 增加系统开销。