---
layout: post
title:  "Swift的autoclosure"
date:   2020-02-14 19:49:44 +0900
categories: ios
---

Swift有个~~不知所谓(划掉)的~~属性(attribute)，叫`@autoclosure`。
官方定义如下

> An autoclosure is a closure that is automatically created to wrap an expression that’s being passed as an argument to a function. It doesn’t take any arguments, and when it’s called, it returns the value of the expression that’s wrapped inside of it. This syntactic convenience lets you omit braces around a function’s parameter by writing a normal expression instead of an explicit closure.
[https://docs.swift.org/swift-book/LanguageGuide/Closures.html](https://docs.swift.org/swift-book/LanguageGuide/Closures.html)

直译的话，autoclosure(自动闭包)就是一个自动构建的闭包。该闭包不接受任何参数，不含任何处理，只返回一个值。
用代码来解释就是

```swift
// 原本你定义一个函数如下
func someFunc(_ para: T) {
    let argument = para
}
// 你这么调用
let arg = T()
someFunc(arg)
```

```swift
// 后来你把形参改成了@autoclosure属性的闭包
func someFunc(_ closure: @autoclosure () -> T) {
    // 于是你在函数内需要执行闭包，才能获取想要的参数
    let argument = closure()
}
// 但是调用函数时，你还是可以直接传递参数
let arg = T()
someFunc(arg)
```

那...那有毛用啊？？？（懵逼表情）

我也觉得确实没毛用啊...😂

如果非要解释起来，我觉得是
1. 鼓励大家使用闭包。闭包的执行是可以控制的，将函数的参数设定为autoclosure后，可以有意识地延迟参数创建的时机。
2. 让**懒人**可以少打两个括号

#### 官方使用场景：

例：`assert(condition:message:file:line:)`函数
可以看到condition和message参数都接受一个autoclosure形式的闭包，在不同优化设定的build下，该函数会决定闭包是否执行。

```swift
/// Performs a traditional C-style assert with an optional message.
///
/// * In playgrounds and `-Onone` builds (the default for Xcode's Debug
///   configuration): If `condition` evaluates to `false`, stop program
///   execution in a debuggable state after printing `message`.
///
/// * In `-O` builds (the default for Xcode's Release configuration),
///   `condition` is not evaluated, and there are no effects.
///
/// * In `-Ounchecked` builds, `condition` is not evaluated, but the optimizer
///   may assume that it *always* evaluates to `true`. Failure to satisfy that
///   assumption is a serious programming error.
public func assert(_ condition: @autoclosure () -> Bool, _ message: @autoclosure () -> String = String(), file: StaticString = #file, line: UInt = #line)
```

#### 强行思考了一些其他使用场景

- 传递的参数可能为某个消耗巨大的函数或者getter方法的返回值，且该参数并不是每次都需要使用

```swift
var hugeString: String {
    var str = "result"
    // 假设这里有很多处理
    str += " is heavy"
    return str
}

var isNecessary: Bool = false
func showStringIfNecessary(_ closure: @autoclosure () -> String) {
    if isNecessary {
        print(closure())
    }
    print("==============")
}

showStringIfNecessary(hugeString)
/*
==============
*/

isNecessary = !isNecessary
showStringIfNecessary(hugeString)
/*
result is heavy
==============
*/
```

- 在当前的队列中，调用某些函数或者getter方法

```swift
var hugeString: String {
    var str = "result"
    // 假设这里有很多处理
    print("[Thread] \(Thread.current)")
    str += " is heavy"
    return str
}

var currentQueue: DispatchQueue = DispatchQueue.global()
func createStringInCurrentQueue(_ closure: @escaping @autoclosure () -> String) {
    currentQueue.async {
        print(closure())
        print("==============")
    }
}

createStringInCurrentQueue(hugeString)
/*
 [Thread] <NSThread: 0x600000acdfc0>{number = 3, name = (null)}
 result is heavy
 ==============
 */

currentQueue = DispatchQueue.main
createStringInCurrentQueue(hugeString)
/*
 [Thread] <NSThread: 0x600000adeb80>{number = 1, name = main}
 result is heavy
 ==============
 */
```

### 用上autoclosure的一个极简logger实现

1. 使用宏`#file`, `#function`等来构建log
2. 出于某些我也不知道为啥的原因，不想在生产环境中调用这些宏
3. 所以用闭包来控制log的构建时机
4. 同时又懒得每次写闭包的括号，所以用上autoclosure

[https://github.com/itsuhi-shu/Logger](https://github.com/itsuhi-shu/Logger)

```swift
import Foundation
import os

func buildLog<T>(_ message: T,
                level: Logger.Level = Logger.level,
                file: String = #file, function: String = #function, line: Int = #line) -> String? {
    guard Logger.Level != .release else { return nil }
    switch level {
    case .verbose:
        let fileName = (file as NSString).lastPathComponent
        return """

        [file:\(fileName)]:[line:\(line)]
        [thread:\(Thread.current)]
        \(message)
        ===========================================
        """
    case .debug:
        return "\(message)"
    case .release:
        return nil
    }
}

func log(_ message: @autoclosure () -> String?) {
    if let message = message() {
        os_log("%@", message)
    }
}

struct Logger {
    enum Level {
        case release
        case debug
        case verbose
    }
    
    static var level: Level = .release
}
```

调用如下

```swift
//
//  AppDelegate.swift
//  ***
//
//  Created by *** on 2020/02/04.
//  Copyright © 2020 ***. All rights reserved.
//

import UIKit

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

    var window: UIWindow?

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {

        Logger.level = .debug
        log(buildLog("App did Launch.", level: .verbose))

        return true
    }

}
```

output

```
2020-02-13 16:15:34.269027+0900 ***[5291:1333819] 
[file:AppDelegate.swift]:[line:18]
[thread:<NSThread: 0x282e26f00>{number = 1, name = main}]
App did Launch.
===========================================
```