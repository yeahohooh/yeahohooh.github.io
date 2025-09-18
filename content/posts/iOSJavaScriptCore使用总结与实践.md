---
title: "iOS JavaScriptCore 使用总结与实践"
date: 2021-05-12T10:10:57+08:00
draft: false
tags: 
  - iOS
author : winsome
scrolltotop : true
toc : true
mathjax : false
---

在 iOS 开发中，JavaScriptCore 提供了一套运行 JavaScript 的引擎，能够让我们在 Native 代码和 JavaScript 之间进行交互。它既可以作为轻量的脚本执行引擎，也能作为桥梁，把原生能力暴露给 JS 使用。

这篇文章主要整理了 JSContext、JSValue、JSVirtualMachine、JSExport 四个核心类的常见用法，以及一些实用技巧。

---

### 常见的 JS 引擎

在移动端或浏览器环境中，常见的 JS 引擎有：

- V8：Google 开发，广泛用于 Android、Chrome。
- JavaScriptCore：苹果开发，Safari 和 WebKit 内核使用的引擎。

在 iOS 原生开发中，我们使用的就是 JavaScriptCore。

---

### JSContext

`JSContext` 代表一个 JavaScript 执行上下文，所有运行过的 JS 都会保存在这个上下文中。我们可以通过它执行 JS 脚本，也能将 Native 方法注入给 JS 调用。

**示例：Hello World**

```
JSContext *jsContext = [[JSContext alloc] init];
JSValue *hello = [jsContext evaluateScript:@"'Hello World!'"];
NSLog(@"%@", [hello toString]);
```
**注入方法**

可以把 Native 的方法注入给 JS：

```
jsContext[@"nslog"] = ^(NSString *message) {
    NSLog(@"%@", message);
};
[jsContext evaluateScript:@"console.log = nslog;"];
[jsContext evaluateScript:@"console.log('Hello from JS')"];
```
**异常处理**

在实际工程中，可以设置 **异常处理器**：

```
jsContext.exceptionHandler = ^(JSContext *context, JSValue *exception) {
    NSLog(@"JS Error: %@", exception);
};
```
---
### JSValue

`JSValue` 是 JS 和 Native 之间的值封装，支持类型互转。

**示例：数值与数组**

```
JSValue *number = [jsContext evaluateScript:@"1+1"];
NSLog(@"%d", [number toInt32]);

JSValue *arr = [jsContext evaluateScript:@"['ha','haha','hahaha']"];
NSLog(@"%@", [arr toArray]);
```
**示例：变量访问**

```
[jsContext evaluateScript:@"var num = 1 + 1"];
JSValue *number = jsContext[@"num"];
NSLog(@"%d", [number toInt32]);

[jsContext evaluateScript:@"var dic = {r:255,g:127,b:80}"];
JSValue *dic = jsContext[@"dic"];
NSLog(@"%@", [dic toDictionary]);
```
---
### JSVirtualMachine

`JSVirtualMachine` 表示一个独立的 JS 执行环境，主要用于并发执行和内存管理。
每个 `JSContext` 必须依赖一个虚拟机，一个虚拟机可以包含多个 `JSContext`。

**示例**

```
JSVirtualMachine *jvm = [[JSVirtualMachine alloc] init];
JSContext *jsContext = [[JSContext alloc] initWithVirtualMachine:jvm];
JSValue *hello = [jsContext evaluateScript:@"'Hello World!'"];
NSLog(@"%@", [hello toString]);
```

**多线程注意事项**

- 不同的 `JSVirtualMachine` 是相互独立的，可以并发执行。
- `JSContext` 不是线程安全的，不能跨线程直接使用。

**示例（两个虚拟机并发执行）**：

```
JSVirtualMachine *jvm1 = [[JSVirtualMachine alloc] init];
JSVirtualMachine *jvm2 = [[JSVirtualMachine alloc] init];
JSContext *context1 = [[JSContext alloc] initWithVirtualMachine:jvm1];
JSContext *context2 = [[JSContext alloc] initWithVirtualMachine:jvm2];

dispatch_queue_t queue = dispatch_queue_create("test", DISPATCH_QUEUE_CONCURRENT);

dispatch_async(queue, ^{
    NSLog(@"%@", [[context1 evaluateScript:@"'thread-1'"] toString]);
});

dispatch_async(queue, ^{
    NSLog(@"%@", [[context2 evaluateScript:@"'thread-2'"] toString]);
});
```
---
### JSExport

`JSExport` 协议可以把 Native 的属性或方法暴露给 JS 使用。

**定义协议**

```
@protocol NativeExport <JSExport>
@property (nonatomic, copy) NSString *str;
- (void)method;
- (void)method2:(NSString *)str;
@end

@interface NativeObject : NSObject <NativeExport>
@end
```
**实现**

```
@implementation NativeObject
- (void)method {
    NSLog(@"call native");
}
- (void)method2:(NSString *)str {
    NSLog(@"%@", str);
}
@end
```

**注入到 JS 环境**

```
jsContext[@"native"] = [[NativeObject alloc] init];
[jsContext evaluateScript:@"native.method()"];
[jsContext evaluateScript:@"native.method2('hello')"];
```

**内存管理**

如果在对象内部注入自己，需要注意释放，避免循环引用：

```
@implementation NativeObject
- (instancetype)init {
    if (self = [super init]) {
        jsContext[@"native"] = self;
    }
    return self;
}
- (void)closeContext {
    self.jsContext[@"native"] = nil;
}
@end
```

---
### JSCore 与 WKWebView 的对比

- JavaScriptCore：适合逻辑计算、脚本扩展，不依赖 UI 渲染。
- WKWebView：适合需要展示 HTML/CSS 的场景，UI 与 JS 强交互时更常用。

---
### 总结

通过 `JSContext`、`JSValue`、`JSVirtualMachine`、`JSExport`，我们可以在 iOS 中构建完整的 JS 执行和交互体系。

- `JSContext`：执行环境
- `JSValue`：值封装
- `JSVirtualMachine`：独立 VM，支持并发与内存管理
- `JSExport`：暴露 Native 接口给 JS

在工程实践中要特别注意：

- 添加 异常处理，避免 JS 报错导致崩溃。
- 注意 线程安全，不要跨线程使用同一个 `JSContext`。
- 管理好 内存释放，避免循环引用。

JavaScriptCore 的优势是 轻量、灵活，非常适合需要 JS 脚本逻辑的场景；但如果涉及 UI 渲染或 H5 页面交互，更推荐使用 WKWebView。
