---
layout: post
title:  "RxSwift4升级至RxSwift5"
date:   2019-11-22 20:44:44 +0900
categories: ios
---

![本栖湖观富士山日出](http://upload-images.jianshu.io/upload_images/1971022-fa9d13cefa89464f.jpg)  

## 起因  

新加入了一个项目，翻了两遍代码后也没新的需求进来，
就这么闲了几天后，负责带我入项目的老哥鬼鬼祟祟地摸到了我座位边上...  
“是不是没活干？”
“...还，还好”
“刚好第三方库有段日子没更新了，要不你都更到最新版本，适配和测试做一下？”
“...！”

## RxSwift4->5  

由于是个小项目，本身用的第三方库也不多，API基本没有变化，理论上是个轻松的活，
直到下面一行diff出现
```git
- github "ReactiveX/RxSwift" "4.5.0"
+ github "ReactiveX/RxSwift" "5.0.1"
```

打开项目Build一下  
哦嚯？！几百个warning  
  
GG……

### Variable被正式废弃并打出`deprecated`警告  

根据[RxSwift开发者的post](https://github.com/ReactiveX/RxSwift/issues/1501#issuecomment-347021795)，
Variable有以下问题
> - it's not a standard cross platform concept so it's out of place in RxSwift target.
> - it doesn't have an extensible counterpart for event management (PublishRelay). It models state only.
> - it is naming is not consistent with *Relay
> - it has an inconsistent memory management model compared to other parts of RxSwift (complete on dealloc)  

大致意思是Variable不是一个跨平台的概念，也不符合Rx的设计标准，因此在RxSwift的未来发展中没有一席之地。  
而[RxSwift的中文文档](https://beeth0ven.github.io/RxSwift-Chinese-Documentation/content/rxswift_core/observable_and_observer/variable.html)也提到

> ...许多开发者滥用 Variable，来构建 重度命令式 系统，而不是 Rx 的 声明式 系统。这对于新手很常见，并且他们无法意识到，这是代码的坏味道。...

对非科班出身程序员的我来说很难产生共鸣...归纳下来就是Variable被生父养母踢出了家门。  
而很明显的是，这个项目组的先人们就是上文所述的滥用Variable的开发者。  
华丽丽地留下了上百条warning

### Variable->BehaviorRelay  

不管是warning内容还是官方文档都确切的注明了用BehaviorRelay来替代Variable，所以没有什么太多需要考虑。  
Variable和BehaviorRelay唯一的区别是Variable被释放时会发出一个Complete事件，而Bahavior没有。
万幸的事本项目没有subscribeVariable的`onComplete`的事件，不需要进行结构性调整。
大部分改动利用Xcode的replace功能，用正则表达式匹配后修改即可。

#### 1. 修改声明
```swift
func setTitle(_ text: Variable<String>) {
    ...
}
```
`Variable<T>`需要改成`BahaviorRelay<T>`  
很简单，把`: Variable`全用`: BahaviorRelay`给replace掉，解决
find: `:( *)Variable`  
replace: `:$1BehaviorRelay`  

#### 2. 修改初始化方法  

```swift
final class AlbumListViewModel: ViewModel {
    var myAlbums = Variable<[PhotoAlbum]>([])
    var memberAlbums = Variable<[PhotoAlbum]>([])
    var numOfAlbum = Variable<[Int]>([])
    ...
}
```
由于声明已经改完了，所以不用在意（本身这个项目内Variable的声明就都是隐式的）  
只需要把
```swift
var myAlbums = Variable<[PhotoAlbum]>([])
```
修改为
```swift
var myAlbums = BehaviorRelay<[PhotoAlbum]>(value: [])
```
用以下正则修改完成
find: `(var|let)(.*)= Variable(.*)\((.*)\)`
replace: `$1$2= BehaviorRelay$3\(value: $4\)`

#### 3. 修改赋值  

```swift
myAlbums.value = albumResponse.myAlbums
memberAlbums.value = albumResponse.memberAlbums
```
由于BehaviorRelay的value属性为只读，现在黄色的warning没了，蹦出一大堆红色的error...
应使用BehaviorRelay的`accept(:)`函数来赋新值，
将以上表达式修改为
```
myAlbums.accept(albumResponse.myAlbums)
```
---

（插入）
有可能出现换行的情况
比如
```swift
licenses.value = licenseItems.compactMap { (licenseItem) -> LicenseItem? in
    ...
}
```
正则小白的作者不知道怎么将特定结尾的字符串除外，于是先匹配出特殊情况，手动修改。
以下情况可能出现换行
find: `(.*)\.value = (.*) (in|\{|\(|\.|\[)$`  

---  

用正则replace一次搞定
找出规律，用以下正则修改
find: `(.*)\.value = (.*)$`
replace: `$1.accept($2)`

#### 4. 修改自增自减  

```swift
var maxSelectedNum = BehaviorRelay<Int>(value: -1)
...
maxSelectedNum.value += 1
```
由于`value`为只读属性，需要设置一个中间变量将`value`拷贝，然后修改中间变量。  
这里新建一个文件，对BehaviorRelay进行扩展，提供两个新的accept函数。
```swift
// BehaviorRelay+Update.swift

import RxRelay

extension BehaviorRelay {

    /// Accept update of current value
    /// - Parameter update: mutate current value in closure
    func acceptUpdate(byMutating update: (inout Element) -> Void) {
        var newValue = value
        update(&newValue)
        accept(newValue)
    }

    /// Accept new value generated from current value
    /// - Parameter update: generate new value from current, and return it
    func acceptUpdate(byNewValue update: (Element) -> Element) {
        accept(update(value))
    }

}
```
然后我们就可以将原代码修改为
```swift
maxSelectedNum.acceptUpdate(byMutating: { $0 += 1 })
// or
maxSelectedNum.acceptUpdate(byNewValue: { $0 + 1 }) //隐式return
```
同样使用正则+replace一键搞定  
find: `(.*)\.value (\+|\-)= ([0-9]+)`
replace: `$1.acceptUpdate(byNewValue: { \$0 $2 $3 })`

#### 5. 修改泛型为数组，字典等的BehaviorRelay的mutating函数  

```swift
var selectedMembers = BehaviorRelay<[Member]>(value: [])
...
selectedMembers.value.append(member)
```
在上面的BehaviorRelay扩展中，已经添加了`acceptUpdate(byMutating:)`函数，只需将上述代码改为
```swift
selectedMembers.value.acceptUpdate(byMutating: { $0.append(member) })
```
即可  
由于这个正则实在不好写，全部手动解决  

#### 6. 添加import  

`RxRelay`被包含在`RxCocoa`module中，但没有被包含在`RxSwift`下，  
而`Variable`是被定义在`RxSwift`中的，  
所以要在需要的地方添加`import RxRelay`

### 其它  

RxSwift 5的其他更新比如，throttle的参数timeInterval改成了`DispatchTimeInterval`类型等，因为在这个项目中影响较小，修改起来比较方便，在此略过不提  
RxSwift5的更新可以参考[这篇文章](https://medium.com/@freak4pc/whats-new-in-rxswift-5-f7a5c8ee48e7)

---

完工！
