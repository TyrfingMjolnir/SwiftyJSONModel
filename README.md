# SwiftyJSONModel
A microframework that helps to use [Swifty JSON](https://github.com/SwiftyJSON/SwiftyJSON) with your custom models in a easy and type-safe way

[![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage) 
[![Build Status](https://travis-ci.org/alickbass/SwiftyJSONModel.svg?branch=master)](https://travis-ci.org/alickbass/SwiftyJSONModel)
[![codecov](https://codecov.io/gh/alickbass/SwiftyJSONModel/branch/master/graph/badge.svg)](https://codecov.io/gh/alickbass/SwiftyJSONModel)

##Motivation:
Let's consider a very simple model:

```swift
struct Person {
    let firstName: String
    let lastName: String
    let age: Int
}
```
Usually, if we want our model to support initialization from `JSON` using `SwiftyJSON` framework, we would have something like this:

```swift
import SwiftyJSON

extension Person {
    init(json: JSON) {
        firstName = json["firstName"].stringValue
        lastName = json["lastName"].stringValue
        age = json["age"].intValue
    }

    var jsonValue: JSON {
        return ["firstName": firstName, "lastName": lastName, "age": age]
    }
}
```
There are several problems with this code:

  * The key for our properties are raw strings and it is easy to make a typo in the key
  * These subscript method will return value even if there is none in the original JSON. But we want to really know when the json is invalid.

###The first solution

The easiest way to take advantage of swift type system is to put all the key in json into `Enum`. Here's an example:

```swift
import SwiftyJSON

extension Person {
    enum PropertyKey: String {
        case firstName, lastName, age
    }
    
    init(json: JSON) {
        firstName = json[PropertyKey.firstName.rawValue].stringValue
        lastName = json[PropertyKey.lastName.rawValue].stringValue
        age = json[PropertyKey.age.rawValue].intValue
    }

    var jsonValue: JSON {
        return [PropertyKey.firstName.rawValue: firstName,
                PropertyKey.lastName.rawValue: lastName,
                PropertyKey.age.rawValue: age]
    }
}
```

But this code still has several problems:

* Although we use enums for our constants, no one restricts us from using some invalid string as a key. Like: `age = json["superRandomKey"].intValue`
* Still we have no clue if the json is actually valid
* We should always manually call the proper method to get proper value. For `int` `intValue`, for `string` - `stringValue`.

#Can we do better?
Yes we can! And here is where our microframework comes to place. 
## Example of usage
Here is the same model but using the `SwiftyJSONModel`:

```swift
import SwiftyJSONModel

extension Person: JSONModelType {
    enum PropertyKey: String {
        case firstName, lastName, age
    }
    
    init(object: JSONObject<PropertyKey>) throws {
        firstName = try object.value(for: .firstName)
        lastName = try object.value(for: .lastName)
        age = try object.value(for: .age)
    }

    var dictValue: [PropertyKey : JSONRepresentable?] {
        return [.firstName: firstName, .lastName: lastName, .age: age]
    }
}

let personJSON = //JSON that we'll use for our model
do {
    let person = try Person(json: personJSON)
    print(person.jsonValue)
} catch let error {
    print(error)
}
```

Although from outside init method stays the same, we now solved all the issues that we had before:

* Keys are now restricted to the `PropertyKey` enum and we will have a compile error if we try to use something different.
* The constructor now `throws` which means that the init will fail if some value or it's type was different from what we expected
* The type of the value for key is now inferred from the property we specify. That means we do not need to have all this boilerplate code with `stringValue` or `intValue`. It will be done for us.

##Recursive errors:
Consider the following code:

```swift
let streetJSON: JSON = [
    "name": "Swifty Boulevard",
    "number": 43,
    "city": [
        "name": "Cocoa",
        "country": [
            "name": 23, // <- Here the name is Int, although we expect String
            "continent": "Africa"
        ]
    ]
]

// Here are our models:
struct Street: JSONObjectInitializable {
    enum PropertyKey: String {
        case name, number, city
    }
    
    let name: String
    let number: Int
    let city: City
    
    init(object: JSONObject<PropertyKey>) throws {
        name = try object.value(for: .name)
        number = try object.value(for: .number)
        city = try object.value(for: .city) // <- this will throw
    }
}

struct City: JSONObjectInitializable {
    enum PropertyKey: String {
        case name, country
    }
    
    let name: String
    let country: Country
    
    init(object: JSONObject<PropertyKey>) throws {
        name = try object.value(for: .name)
        country = try object.value(for: .country) // <- this will throw
    }
}

struct Country: JSONObjectInitializable {
    enum PropertyKey: String {
        case name, continent
    }
    
    let name: String
    let continent: String
    
    init(object: JSONObject<PropertyKey>) throws {
        name = try object.value(for: .name) // <- this will throw
        continent = try object.value(for: .continent)
    }
}

do {
    let street = try Street(json: streetJSON)
} catch let error {
    print(error) // prints "[city][country][name]: Invalid element"
}
```

The code `Street(json: streetJSON)` will throw error as we expect the country's name to be a `String`, but we received `Int`. The library will throw a verbose error that will show the full path to the invalid property.

In the given example `print` will give the following result: 

`[city][country][name]: Invalid element`

This feature is really powerful to debug the json that has a nested structure, especially in the process of development 😉


