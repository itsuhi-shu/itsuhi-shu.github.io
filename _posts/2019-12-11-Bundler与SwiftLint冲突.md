---
layout: post
title:  "Bundler与SwiftLint冲突"
date:   2019-12-11 19:23:44 +0900
categories: ios
---

**个人memo用文章**

同时使用SwiftLint和Bundler时，务必注意把在`.swiftlint.yml`中把vendor文件夹除外！！

Xcode中莫名出现了一堆fastlane的报错，懵逼了半天。
把build settings和build phase中的path翻了个底朝天也没发现为什么vendor文件下的东西会编译报错。

最后终于想起来把报错信息在Log中打印一遍。

```
Linting 'ScreengrabfileProtocol.swift' (306/332)
/Users/.../dev/.../vendor/bundle/ruby/2.6.0/gems/fastlane-2.137.0/fastlane/swift/RubyCommand.swift:60:57: warning: Colon Violation: Colons should be next to the identifier when specifying a type and next to the key in dictionary literals. (colon)
/Users/.../dev/.../vendor/bundle/ruby/2.6.0/gems/fastlane-2.137.0/fastlane/swift/RubyCommand.swift:114:26: error: Empty Count Violation: Prefer checking `isEmpty` over comparing `count` to zero. (empty_count)
```

给个中指先...

```yml
excluded:
  - Carthage
  - Pods
  ...
  - vendor
```
在`.swiftlint.yml`文件中除外后搞定