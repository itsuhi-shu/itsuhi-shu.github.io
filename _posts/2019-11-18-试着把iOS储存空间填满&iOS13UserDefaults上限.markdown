---
layout: post
title:  "试着把iOS储存空间填满&iOS13UserDefaults上限"
date:   2019-11-18 15:37:44 +0900
categories: ios
---

做iOS开发做到文件储存这一块的时候，面对各种各样的`try`语句，总会想如果真的储存满了会发生什么情况。
于是脑洞大开，就做个把iOS储存空间塞满的app，以满足测试场景。

### 填满实现方案

实现方案非常简单，就是用各种空数据堆成文件，然后用`FileManager`储存进sandbox

1. 利用`volumeAvailableCapacityForImportantUsageKey`可以查询到可以使用的储存空间（单位为字节）
⬇️苹果官方示例代码

```swift
let fileURL = URL(fileURLWithPath:"/")
do {
    let values = try fileURL.resourceValues(forKeys: [.volumeAvailableCapacityForImportantUsageKey])
    if let capacity = values.volumeAvailableCapacityForImportantUsage {
        print("Available capacity for important usage: \(capacity)")
    } else {
        print("Capacity is unavailable")
    }
} catch {
    print("Error retrieving capacity: \(error.localizedDescription)")
}
```

2. 把字节转换为GB, MB, KB
   - Swift可以把函数作为参数传递，所以用一个包含了运算符和乘数的元组作为单位转换函数的返回值
   - enum的类型取为Int，从bit开始rawValue递增，每升一级乘以1024，然后以此为基础设计获得乘数的递归函数
   - 两个assertionFailure的地方，从这个程序设计的角度来看是不可能进入的，参照喵神的文章
[关于 Swift Error 的分类](https://onevcat.com/2017/10/swift-error-category/)
这里应为**logic failure**, 用assertionFailure让程序在debug时进入后直接崩溃

```swift
struct DataAllocator {

    ...

    enum Unit: Int, CustomStringConvertible {
        case bits = 0
        case bytes
        case kilobytes
        case megabytes
        case gigabytes

        var description: String {
            switch self {
            case .bits:
                return "bits"
            case .bytes:
                return "bytes"
            case .kilobytes:
                return "kilobytes"
            case .megabytes:
                return "megabytes"
            case .gigabytes:
                return "gigabytes"
            @unknown default:
                return "unknown"
            }
        }
    }

    static func size(of size: Int, from fromUnit: Unit, to toUnit: Unit) -> Int {
        let (oprt, multiplier) = conversion(from: fromUnit, to: toUnit)
        return oprt(size, multiplier)
    }

    static func conversion(from fromUnit: Unit, to toUnit: Unit) -> (operator: (Int, Int) -> Int, multiplier: Int) {
        switch fromUnit.rawValue {
        case 0...toUnit.rawValue:
            return (/, multiplier(from: toUnit, to: fromUnit))
        case (toUnit.rawValue + 1)...:
            return (*, multiplier(from: fromUnit, to: toUnit))
        default:
            assertionFailure("unable to convert from \(fromUnit) to \(toUnit)")
            return (*, 1)
        }
    }

    private static func multiplier(from fromUnit: Unit, to toUnit: Unit) -> Int {
        if fromUnit == toUnit { return 1 }
        guard fromUnit.rawValue > toUnit.rawValue,
            let lowerUnit = Unit(rawValue: fromUnit.rawValue - 1) else {
                assertionFailure("unable to get multiplier from \(fromUnit) to \(toUnit)")
                return 1
        }
        return 1024 * multiplier(from: lowerUnit, to: toUnit)
    }
}
```  
  
```swift
private func display(capacity: Int64) {
    let total = Int(capacity)
    var remainder = total
    let gigas = DataAllocator.size(of: remainder, from: .bytes, to: .gigabytes)
    remainder -= DataAllocator.size(of: gigas, from: .gigabytes, to: .bytes)
    let megas = DataAllocator.size(of: remainder, from: .bytes, to: .megabytes)
    remainder -= DataAllocator.size(of: megas, from: .megabytes, to: .bytes)
    let kilos = DataAllocator.size(of: remainder, from: .bytes, to: .kilobytes)
    infos = [("Total", total), ("GB", gigas), ("MB", megas), ("KB", kilos)]
    // refreshInfos()
}
```

3. 申请特定大小的内存，然后用写入文件，用FileManager存入sandbox中，已知UInt8大小为一字节...
    - 由于iOS设备内存基本都在3G以下，还有很多1G，所以单次文件储存控制在512M以下
    - 使用for语句需加上autoreleasepool，否则for语句内不会自动释放内存，内存爆炸
  
```swift
struct DataAllocator {

    typealias Byte = [UInt8]

    private(set) var unit: Unit

    init(unit: Unit) {
        self.unit = unit
    }

    func allocateMemory(of size: Int) -> Byte {
        let buffer = Byte(repeating: 0,
                          count: DataAllocator.size(of: size,
                                                    from: unit,
                                                    to: .bytes))
        return buffer
    }

    static func allocateMemory(of size: Int, unit: Unit) -> Byte {
        let allocator = DataAllocator(unit: unit)
        return allocator.allocateMemory(of: size)
    }

    ...
}
```  
  
```swift
static func fileStorage(data: [UInt8], namePrefix: String, nameSuffix: String) throws -> String {
    let content = Data(bytes: data, count: data.count)
    let fileManager = FileManager.default
    let documentDirectory = try fileManager.url(for: .documentDirectory, in: .userDomainMask, appropriateFor:nil, create:false)
    let fileName = namePrefix + "_filestorage_" + nameSuffix
    let fileURL = documentDirectory.appendingPathComponent(fileName)
    try content.write(to: fileURL)
    return fileName
}
```  
  
```swift
private func fulfillStorage() {
    let gigas = infos[1].detail
    let megas = infos[2].detail
    let kilos = infos[3].detail
    let alertTitle = "Store"

    func fileStorage(unit: DataAllocator.Unit, size: Int, frenquency: Int) {
        do {
            try FileStorage.fileStorage(unit: unit, size: size, frequency: frenquency)
        } catch {
            print(error.localizedDescription)
        }
    }

    DispatchQueue.global().async {
        if gigas > 0 {
            for idx in 1...(gigas * 2) {
                autoreleasepool {
                    fileStorage(unit: .megabytes, size: 512, frenquency: idx)
                }
            }
        }
        if megas > 0 {
            for idx in 1...megas {
                autoreleasepool {
                    fileStorage(unit: .megabytes, size: 1, frenquency: idx)
                }
            }
        }
        if kilos > 0 {
            for idx in 1...kilos {
                autoreleasepool {
                    fileStorage(unit: .kilobytes, size: 1, frenquency: idx)
                }
            }
        }
    }
}
```

#### 填满iOS储存空间app完整代码
https://github.com/itsuhi-shu/FulfillStorage

#### 遇到的问题

很显然申请到的内存存入文件后所占用的空间并不完全等于内存大小...虽然能够想到这问题，但是本菜鸟非科班出身无法搞清个所以然来，而且差别并不大，所以忽略不计

### iOS13的UserDefaults储存上限警告（4MB）
（虽然和以上内容并无太大关联，但是正好同时发现所以一起贴出来）

iOS的UserDefaults理论上可以储存和可用空间等大的内容，然而从iOS13开始，UserDefaults使用超过4M后每次尝试储存都会出现警告

`2019-11-14 13:40:35.383876+0900 TestUserDefaults[21230:7116172] [User Defaults] CFPrefsPlistSource<0x600002bc0580> (Domain: com.yifei-zhou.TestUserDefaults, User: kCFPreferencesCurrentUser, ByHost: No, Container: (null), Contents Need Refresh: Yes): Attempting to store >= 4194304 bytes of data in CFPreferences/NSUserDefaults on this platform is invalid. This is a bug in TestUserDefaults or a library it uses`

`2019-11-14 13:40:35.384214+0900 TestUserDefaults[21230:7116172] [User Defaults] CFPrefsPlistSource<0x600002bc0580> (Domain: com.yifei-zhou.TestUserDefaults, User: kCFPreferencesCurrentUser, ByHost: No, Container: (null), Contents Need Refresh: No): Transitioning into direct mode`

嘛，一般也不会往UserDefaults里存那么多东西就是了
