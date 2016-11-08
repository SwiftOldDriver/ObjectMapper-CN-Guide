# ObjectMapper-CN-Guide
ObjectMapper 是一个使用 Swift 编写的用于 model 对象（类和结构体）和 JSON  之间转换的框架。

- [特性](#features)
- [基础使用方法](#the-basics)
- [Mapping Nested Objects](#easy-mapping-of-nested-objects)
- [Custom Transformations](#custom-transforms)
- [Subclassing](#subclasses)
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