---
layout: post
title:  "使用Mirror自动比较Swift Class, Struct的尝试"
date:   2020-01-20 12:09:44 +0900
categories: ios
---

#### 遇到的问题

1. 上线了两年的项目，没有一丁点测试代码，突然说要开始补单元测试
2. 上百个Model，有`struct`，有`class`，全都没有遵循`Equatable`协议...要比较两个Model是否相同，需要逐条比较Model的property（属性）

#### 想做的事情

能否以最小的成本让上百个Model在测试中可以自动比较

## 使用Mirror获取property list

#### 动态地获取property list

Objective-C强大的运行时系统使得开发者可以在运行时获取class的property list。
[https://stackoverflow.com/questions/754824/get-an-object-properties-list-in-objective-c](https://stackoverflow.com/questions/754824/get-an-object-properties-list-in-objective-c)
因此碰到这个问题时，我首先想的是，是否Swift也可以用类似`class_copyPropertyList`这样的API获取对象的property list。
但是发现想多了...Swift是一门静态语言，在Swift官方主页的介绍中就强调了

> Swift code is safe by design

这样的表述。在运行时动态地获取和篡改对象的property显然不是Swift所推崇的。
  
不过Swift还是很体贴地提供了一个工具：Mirror，来满足开发者的一些运行时需求。

#### Mirror的定义

> A mirror describes the parts that make up a particular instance, such as the instance’s stored properties, collection or tuple elements, or its active enumeration case. Mirrors also provide a “display style” property that suggests how this mirror might be rendered.

Mirror描述了一个特定实例的结构，
- 实例为class或struct，则描述该实例的储存属性 (stored properties)
- 实例为collection或tuple (元组) ，则描述其elements (元素)
- 实例为enumeration，则描述其当前case

> Playgrounds and the debugger use the Mirror type to display representations of values of any type. For example, when you pass an instance to the dump(_:_:_:_:) function, a mirror is used to render that instance’s runtime contents.

关于Mirror的用途，官方文档提到Mirror主要是在Playgrounds或debug时使用，Swift的调试用函数`dump(_:name:indent:maxDepth:maxItems:)`即是使用Mirror来反射实例。
[https://developer.apple.com/documentation/swift/1539127-dump](https://developer.apple.com/documentation/swift/1539127-dump)
虽然没有明确禁止在生产环境中使用Mirror，但是Mirror强大的运行时特性注定了它不是一个高效的选择（参考自喵神这篇文章[https://onevcat.com/2018/03/swift-meta/](https://onevcat.com/2018/03/swift-meta/)）。
嘛，单元测试里拿来用一下应该问题不大吧哈哈哈哈h...

#### Mirror的基础用法

光说不练假把式，我们来实际使用Mirror试试。
Mirror的使用方法非常简单，使用Mirror的`init(reflecting:)`初始化方法，传入想要反射的对象，就能生成反射对象的Mirror实例。

```swift
class AClass {
    let storedProperty: String
    var computedProperty: String {
        return "hi"
    }
    init(title: String) {
        storedProperty = title
    }
}

let aObject = AClass(title: "a")
let aObjectMirror = Mirror(reflecting: aObject)
```

调用Mirror实例的`children`属性可以得到一个Mirror.Child的collection，而Mirror.Child则是一个包含`label`和`value`两个元素的元组。
`label`在反射对象为class或struct时为对象的属性名，`value`则是属性值
注意此处`label`为Optional，在反射对象为collection(array, set, dictionary)时，`label`为`nil`，`value`为collection的元素，下文会再次提及。

```swift
typealias Mirror.Children = AnyCollection<Mirror.Child>
typealias Mirror.Child = (label: String?, value: Any)
```

遍历`aObjectMirror`的`children`并打印，可以看到`AClass`的储存属性都被反射，而计算属性没有被反射。

```swift
aObjectMirror.children.forEach { print($0) }
/* output:
 (label: Optional("storedPropertyStr"), value: "a")
 (label: Optional("storedPropertyInt"), value: 1)
 */
```

使用Mirror反射class实例的基本用法如上所示，按官方文档的说法，Mirror还可以作用于collection, tuple, enumeration。那具体反射后得到的Mirror是什么样的，我们一个个把玩下
[https://github.com/itsuhi-shu/PropertyEquatable/blob/master/MirrorPlayground.playground/Contents.swift](https://github.com/itsuhi-shu/PropertyEquatable/blob/master/MirrorPlayground.playground/Contents.swift)

因篇幅限制，挑一些比较在意的内容总结：

- class, struct的计算属性不会被反射
- collection(array, set, dictionary)反射后其child的`label`为`nil`
  -) array, set的`value`为其各个元素
  -) dictionary的`value`为其单个键值对构成的元组`(key: Hashable, value: Any)`

```swift
let aDictionary = ["key1": "a", "key2": "b", "key3": "c"]
let aDictionaryMirror = Mirror(reflecting: aDictionary)
print(aDictionaryMirror.displayStyle!) // dictionary
aDictionaryMirror.children.forEach { print($0) }
/*
 (label: nil, value: (key: "key2", value: "b"))
 (label: nil, value: (key: "key1", value: "a"))
 (label: nil, value: (key: "key3", value: "c"))
 */
```

- tuple的label若有定义标签名则为标签名，若没有则为`.n`（n为该元素的序列）

```swift
let aTuple = ("a", labeled: 2, 9)
let aTupleMirror = Mirror(reflecting: aTuple)
print(aTupleMirror.displayStyle!) // tuple
aTupleMirror.children.forEach { print($0) }
/*
 (label: Optional(".0"), value: "a")
 (label: Optional("labeled"), value: 2)
 (label: Optional(".2"), value: 9)
 */
```

- enum只能反射associated中储存的属性
  
```swift
enum AEnum {
    case first
    case seconde
}
enum AEnumWithAssociatedValues {
    case first
    case second(with: String)
    case third(title: String, Value: Int, complete: (() -> Void)?)
}

let aEnum = AEnum.seconde
let aEnumMirror = Mirror(reflecting: aEnum)
print(aEnumMirror.displayStyle!) // enum
aEnumMirror.children.forEach { print($0) }
/*
 */
print(aEnumMirror.children.count) // 0

let aEnumWithAssociatedValues = AEnumWithAssociatedValues.third(title: "a",
                                                                Value: 1,
                                                                complete: nil)
let aEnumWithAssociatedValuesMirror = Mirror(reflecting: aEnumWithAssociatedValues)
print(aEnumWithAssociatedValuesMirror.displayStyle!) // enum
aEnumWithAssociatedValuesMirror.children.forEach { print($0) }
/*
 (label: Optional("third"), value: (title: "a", Value: 1, complete: nil))
 */
```  

## 利用Mirror反射来自动比较Model

了解了Mirror的基本用法，现在该思考如何将Mirror运用在开发中。
在上面提到，笔者想要写最少的代码来实现Model的比较，在有了Mirror之后，接下来很自然而言地就想到要定义一个protocol（协议），通过protocol的extension（扩展协议）来为Model添加一个比较方法。
一开始曾试过用protocol extension来让Model遵循`Equatable`协议，并添加对`==`的实现。但后来突然发现`NSObject`都遵循了`Equatable`协议，且其`==`实现为单纯的地址比较（一部分子类override了）。而protocol extension不能override类已实现的方法。所以只能另外提供一个比较函数。

最终的实现如下：

```swift
protocol PropertyEquatable {}
extension PropertyEquatable {
    // Do not use `==` because extension can NOT override methods,
    // and most NSObjects conforms to `Equatable` by implementing `==` simply compare their addresses.
    static func ~= (lhs: Self, rhs: Self) -> Bool {
        func _recursiveCompareElements(lhs: Any, rhs: Any) -> Bool {
            guard type(of: lhs) == type(of: rhs) else { return false }

            let lMir = Mirror(reflecting: lhs)
            let rMir = Mirror(reflecting: rhs)

            return lMir.children.elementsEqual(rMir.children) { (lElm, rElm) -> Bool in
                guard let lKey = lElm.label,
                    let rKey = rElm.label,
                    lKey == rKey else {
                        return false
                }

                // MARK: Collection
                // Arrays, Sets, Dictionaries are all Hashable, and can easily fall down to AnyHashable.
                // But if the Elements are MirrorEquatable, we need to compare the Elements using our methods.

                // Arrays
                if let lArr = lElm.value as? Array<PropertyEquatable>,
                    let rArr = rElm.value as? Array<PropertyEquatable> {
                    return lArr.elementsEqual(rArr) { _recursiveCompareElements(lhs: $0, rhs: $1) }
                }

                // Sets Elements must be Hashable, we don't need to consider about this case.

                // Dictionaries
                if let lDic = lElm.value as? [AnyHashable: PropertyEquatable],
                    let rDic = rElm.value as? [AnyHashable: PropertyEquatable] {
                    guard lDic.count == rDic.count else { return false }
                    for key in lDic.keys {
                        guard let lVal = lDic[key], let rVal = rDic[key] else { return false }
                        if !_recursiveCompareElements(lhs: lVal, rhs: rVal) {
                            return false
                        }
                    }
                    return true
                }

                // MARK: AnyHashable
                if let lVal = lElm.value as? AnyHashable,
                    let rVal = rElm.value as? AnyHashable {
                    return lVal == rVal
                }

                // MARK: Classes, Structs and others that do not match the conditions above
                return _recursiveCompareElements(lhs: lElm.value, rhs: rElm.value)
            }
        }

        return _recursiveCompareElements(lhs: lhs, rhs: rhs)
    }
}
```

主要想法如下：
1. 定义一个协议，并扩展该协议实现对比方法。
2. 若该类的属性的类也遵循该协议，则递归地往下挖掘并对比。
3. 直到挖到属性不遵循该协议且为Hashable之后，调用`==`方法比对属性。
4. 考虑属性为collection的情况。

完成！！
[https://github.com/itsuhi-shu/PropertyEquatable](https://github.com/itsuhi-shu/PropertyEquatable)