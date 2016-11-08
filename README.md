# ObjectMapper-CN-Guide
ObjectMapper 是一个使用 Swift 编写的用于 model 对象（类和结构体）和 JSON  之间转换的框架。

- [特性](#features)
- [基础使用方法](#the-basics)
- [映射嵌套对象](#easy-mapping-of-nested-objects)
- [自定义转换规则](#custom-transforms)
- [继承](#subclasses)
- [Generic Objects](#generic-objects)
- [Mapping Context](#mapping-context)
- [ObjectMapper + Alamofire](#objectmapper--alamofire) 
- [ObjectMapper + Realm](#objectmapper--realm)
- [To Do](#to-do)
- [Contributing](#contributing)
- [Installation](#installation)

# 特性:
- 把 JSON 映射成对象 
- 把对象映射 JSON
- 支持嵌套对象 (单独的成员变量、在数组或字典中都可以)
- 在转换过程支持自定义规则
- 支持结构体（ Struct ）
- [Immutable support](#immutablemappable-protocol-beta) (目前还在 beta )

# 基础使用方法
为了支持映射，类或者结构体只需要实现```Mappable```协议。这个协议包含以下方法：
```swift
init?(map: Map)
mutating func mapping(map: Map)
```
ObjectMapper使用自定义的```<-``` 运算符来声明成员变量和 JSON 的映射关系。
```swift
class User: Mappable {
    var username: String?
    var age: Int?
    var weight: Double!
    var array: [AnyObject]?
    var dictionary: [String : AnyObject] = [:]
    var bestFriend: User?                       // 嵌套的 User 对象
    var friends: [User]?                        // Users 的数组
    var birthday: NSDate?

    required init?(map: Map) {

    }

    // Mappable
    func mapping(map: Map) {
        username    <- map["username"]
        age         <- map["age"]
        weight      <- map["weight"]
        array       <- map["arr"]
        dictionary  <- map["dict"]
        bestFriend  <- map["best_friend"]
        friends     <- map["friends"]
        birthday    <- (map["birthday"], DateTransform())
    }
}

struct Temperature: Mappable {
    var celsius: Double?
    var fahrenheit: Double?

    init?(map: Map) {

    }

    mutating func mapping(map: Map) {
        celsius 	<- map["celsius"]
        fahrenheit 	<- map["fahrenheit"]
    }
}
```

一旦你的对象实现了 `Mappable`, ObjectMapper就可以让你轻松的实现和 JSON 之间的转换。

把 JSON 字符串转成 model 对象：

```swift
let user = User(JSONString: JSONString)
```

把一个 model 转成 JSON 字符串：

```swift
let JSONString = user.toJSONString(prettyPrint: true)
```

也可以使用`Mapper.swift`类来完成转换（这个类还额外提供了一些函数来处理一些特殊的情况：

```swift
// 把 JSON 字符串转成 Model
let user = Mapper<User>().map(JSONString: JSONString)
// 根据 Model 生成 JSON 字符串
let JSONString = Mapper().toJSONString(user, prettyPrint: true)
```

ObjectMapper支持以下的类型映射到对象中：

- `Int`
- `Bool`
- `Double`
- `Float`
- `String`
- `RawRepresentable` (枚举)
- `Array<AnyObject>`
- `Dictionary<String, AnyObject>`
- `Object<T: Mappable>`
- `Array<T: Mappable>`
- `Array<Array<T: Mappable>>`
- `Set<T: Mappable>` 
- `Dictionary<String, T: Mappable>`
- `Dictionary<String, Array<T: Mappable>>`
- 以上所有的 Optional 类型
- 以上所有的隐式强制解包类型（Implicitly Unwrapped Optional）

## `Mappable` 协议

#### `mutating func mapping(map: Map)` 
所有的映射最后都会调用到这个函数。当解析 JSON 时，这个函数会在对象创建成功后被执行。当生成 JSON 时就只有这个函数会被对象调用。

#### `init?(map: Map)` 
这个可失败的初始化函数是 ObjectMapper 创建对象的时候使用的。开发者可以通过这个函数在映射前校验 JSON 。如果在这个方法里返回 nil 就不会执行 `mapping` 函数。可以通过传入的保存着 JSON 的  `Map` 对象进行校验：

```swift
required init?(map: Map){
	// 检查 JSON 里是否有一定要有的 "name" 属性
	if map.JSONDictionary["name"] == nil {
		return nil
	}
}
```

## `StaticMappable` 协议
`StaticMappable` 是 `Mappable` 之外的另一种选择。 这个协议可以让开发者通过一个静态函数初始化对象而不是通过 `init?(map: Map)`。

注意: `StaticMappable` 和 `Mappable` 都继承了 `BaseMappable` 协议。 `BaseMappable` 协议声明了 `mapping(map: Map)` 函数。

#### `static func objectForMapping(map: Map) -> BaseMappable?` 
ObjectMapper 使用这个函数获取对象后进行映射。开发者需要在这个函数里返回一个实现 `BaseMappable` 对象的实例。这个函数也可以用于：

- 在对象进行映射前校验 JSON 
- 提供一个缓存过的对象用于映射
- 返回另外一种类型的对象（当然是必须实现了 BaseMappable）用于映射。比如你可能通过检查 JSON 推断出用于映射的对象 ([看这个例子](https://github.com/Hearst-DD/ObjectMapper/blob/master/ObjectMapperTests/ClassClusterTests.swift#L62))。

如果你需要在 extension 里实现 ObjectMapper，你需要选择这个协议而不是 `Mappable` 。

# 轻松映射嵌套对象

ObjectMapper 支持使用点语法来轻松实现嵌套对象的映射。比如有如下的 JSON 字符串：

```json
"distance" : {
     "text" : "102 ft",
     "value" : 31
}
```
你可以通过这种写法直接访问到嵌套对象：

```swift
func mapping(map: Map) {
    distance <- map["distance.value"]
}
```
嵌套的键名也支持访问数组中的值。如果有一个返回的 JSON 是一个包含 distance 的数组，可以通过这种写法访问：

```
distance <- map["distances.0.value"]
```
如果你的键名刚好含有 `.` 符号，你需要特别声明关闭上面提到的获取嵌套对象功能：

```swift
func mapping(map: Map) {
    identifier <- map["app.identifier", nested: false]
}
```
如果刚好有嵌套的对象的键名还有 `.` ,可以在中间加入一个自定义的分割符（[#629](https://github.com/Hearst-DD/ObjectMapper/pull/629)）:
```swift
func mapping(map: Map) {
    appName <- map["com.myapp.info->com.myapp.name", delimiter: "->"]
}
```
这种情况的 JSON 是这样的：

```json
"com.myapp.info" : {
     "com.myapp.name" : "SwiftOldDriver"
}
```

# 自定义转换规则
ObjectMapper 也支持在映射时自定义转换规则。如果要使用自定义转换，创建一个 tuple（元祖）包含 ```map["field_name"]``` 和你要使用的变换放在 ```<-``` 的右边：

```swift
birthday <- (map["birthday"], DateTransform())
```
当解析 JSON 时上面的转换会把 JSON 里面的 Int 值转成一个 NSDate ，如果是对象转为 JSON 时，则会把 NSDate 对象转成 Int 值。

只要实现```TransformType``` 协议就可以轻松的创建自定义的转换规则：

```swift
public protocol TransformType {
    associatedtype Object
    associatedtype JSON

    func transformFromJSON(_ value: Any?) -> Object?
    func transformToJSON(_ value: Object?) -> JSON?
}
```

### TransformOf
大多数情况下你都可以使用框架提供的转换类 ```TransformOf``` 来快速的实现一个期望的转换。 ```TransformOf``` 的初始化需要两个类型和两个闭包。两个类型声明了转换的目标类型和源类型，闭包则实现具体转换逻辑。

举个例子，如果你想要把一个 JSON 字符串转成 Int ，你可以像这样使用 ```TransformOf``` ：

```swift
let transform = TransformOf<Int, String>(fromJSON: { (value: String?) -> Int? in 
    // 把值从 String? 转成 Int?
    return Int(value!)
}, toJSON: { (value: Int?) -> String? in
    // 把值从 Int? 转成 String?
    if let value = value {
        return String(value)
    }
    return nil
})

id <- (map["id"], transform)
```
这是一种更省略的写法：

```swift
id <- (map["id"], TransformOf<Int, String>(fromJSON: { Int($0!) }, toJSON: { $0.map { String($0) } }))
```
# 继承

实现了  ```Mappable``` 协议的类可以容易的被继承。当继承一个 mappable 的类时，使用这样的结构：

```swift
class Base: Mappable {
	var base: String?
	
	required init?(map: Map) {

	}

	func mapping(map: Map) {
		base <- map["base"]
	}
}

class Subclass: Base {
	var sub: String?

	required init?(map: Map) {
		super.init(map)
	}

	override func mapping(map: Map) {
		super.mapping(map)
		
		sub <- map["sub"]
	}
}
```

注意确认子类中的实现调用了父类中正确的初始化器和映射函数。