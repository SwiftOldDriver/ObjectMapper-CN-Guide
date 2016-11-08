# ObjectMapper-CN-Guide
ObjectMapper 是一个使用 Swift 编写的用于 model 对象（类和结构体）和 JSON  之间转换的框架。

- [特性](#features)
- [The Basics](#the-basics)
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
- 支持嵌套对象 (在数组或字典中)
- 在转换过程支持自定义规则
- 支持结构体（ Struct ）
- [Immutable support](#immutablemappable-protocol-beta) (目前还在 beta )

# 基础使用方法
To support mapping, a class or struct just needs to implement the ```Mappable``` protocol which includes the following functions:
```swift
init?(map: Map)
mutating func mapping(map: Map)
```
ObjectMapper uses the ```<-``` operator to define how each member variable maps to and from JSON.

```swift
class User: Mappable {
    var username: String?
    var age: Int?
    var weight: Double!
    var array: [AnyObject]?
    var dictionary: [String : AnyObject] = [:]
    var bestFriend: User?                       // Nested User object
    var friends: [User]?                        // Array of Users
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

Once your class implements `Mappable`, ObjectMapper allows you to easily convert to and from JSON. 

Convert a JSON string to a model object:
```swift
let user = User(JSONString: JSONString)
```

Convert a model object to a JSON string:
```swift
let JSONString = user.toJSONString(prettyPrint: true)
```

Alternatively, the `Mapper.swift` class can also be used to accomplish the above (it also provides extra functionality for other situations):
```
// Convert JSON String to Model
let user = Mapper<User>().map(JSONString: JSONString)
// Create JSON String from Model
let JSONString = Mapper().toJSONString(user, prettyPrint: true)
```

ObjectMapper can map classes composed of the following types:
- `Int`
- `Bool`
- `Double`
- `Float`
- `String`
- `RawRepresentable` (Enums)
- `Array<AnyObject>`
- `Dictionary<String, AnyObject>`
- `Object<T: Mappable>`
- `Array<T: Mappable>`
- `Array<Array<T: Mappable>>`
- `Set<T: Mappable>` 
- `Dictionary<String, T: Mappable>`
- `Dictionary<String, Array<T: Mappable>>`
- Optionals of all the above
- Implicitly Unwrapped Optionals of the above