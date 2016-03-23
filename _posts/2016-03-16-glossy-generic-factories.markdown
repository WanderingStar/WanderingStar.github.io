---
layout: post
title:  "Glossy Generic Factories"
date:   2016-03-16 23:46:29 -0500
tags: swift gloss json generics
summary: >
  Dealing with JSON collections of heterogeneous subtypes,
  but more generally.

---
[Last time][last] we looked at creating one-off factories that we can use to 
parse collections of subclass objects with [Gloss][gloss]. But it's a drag
to have to put the Factory boilerplate in multiple places and define
separate decoders for each group of types.

You can download a [playground here](/uploads/GlossyGenericFactories.playground.zip).

This time we're going to take a step back and use generics to abstract out
the Factory...

Here is our Factory. It knows the keyPath to look at to find the
type identifier in a JSON block, and it has a table of subtypes to
create based on what it finds there.

{% highlight swift %}
class Factory<T: Decodable> {
    let typeKeyPath: String
    var subtypes = [String: T.Type]()
    
    init(typeKeyPath: String) {
        self.typeKeyPath = typeKeyPath
    }
    
    func register(typeKey: String, asType: T.Type) {
        subtypes[typeKey] = asType
    }
    
    func make(json: JSON) -> T? {
        if let type = json.valueForKeyPath(typeKeyPath) as? String,
            subtype = subtypes[type] {
                return subtype.init(json: json)
        }
        return nil
    }
}
{% endhighlight %}

Our subtypes should implement this protocol, which lets us
discover the factory that we should use to make them:

{% highlight swift %}
protocol FactoryDecodable: Decodable {
    typealias Output: Decodable
    static var factory: Factory<Output> { get }
}
{% endhighlight %}

Ideally, we'd specify that the Output of the factory is a
FactoryDecodable, not just a Decodable, but we can't use the protocol
as a requirement of itself.

Our supertype defines the factory itself.

{% highlight swift %}
class Shape: FactoryDecodable {
    static var factory = Factory<Shape>(typeKeyPath: "type")
    
    var area: Double { return 0 }
    
    required init?(json: JSON) {
        // base class does nothing
    }
    
}
{% endhighlight %}

There's still nothing special in our subtypes...

{% highlight swift %}
class Square: Shape {
    let side: Double
    override var area: Double { return side * side }
    
    required init?(json: JSON) {
        if let side: Double = "side" <~~ json {
            self.side = side
            super.init(json: json)
        } else {
            return nil
        }
    }
}

class Circle: Shape {
    let radius: Double
    override var area: Double { return M_PI * radius * radius }
    
    required init?(json: JSON) {
        if let radius: Double = "radius" <~~ json {
            self.radius = radius
            super.init(json: json)
        } else {
            return nil
        }
    }
}
{% endhighlight %}

Our subtypes must be registered with the supertype's Factory.

{% highlight swift %}
Shape.factory.register("square", asType: Square.self)
Shape.factory.register("circle", asType: Circle.self)
{% endhighlight %}

And now the Glossy part... Decoders that look for FactoryDecodables
and collections of them:

{% highlight swift %}
extension Decoder {
    
    static func decodeWithFactory<T: FactoryDecodable>(key: String, json: JSON) -> T? {
        if let keyed = json.valueForKeyPath(key) as? JSON {
            return T.factory.make(keyed) as? T
        }
        return nil
    }
    
    static func decodeArrayWithFactory<T: FactoryDecodable>(key: String, json: JSON) -> [T]? {
        guard let array = json.valueForKeyPath(key) as? [JSON] else { return nil }
        return array.flatMap { T.factory.make($0) as? T }
    }
    
    static func decodeDictionaryWithFactory<T: FactoryDecodable>(key: String, json: JSON) -> [String: T]? {
        guard let dict = json.valueForKeyPath(key) as? [String: JSON] else { return nil }
        var made = [String: T]()
        for (key, value) in dict {
            if let element = T.factory.make(value) as? T {
                made[key] = element
            }
        }
        return made
    }
}
{% endhighlight %}

And operators to make those nicer to use... 

{% highlight swift %}
infix operator <~*~ { associativity left precedence 150 }


func <~*~ <T: FactoryDecodable>(key: String, json: JSON) -> T? {
    return Decoder.decodeWithFactory(key, json: json)
}

func <~*~ <T: FactoryDecodable>(key: String, json: JSON) -> [T]? {
    return Decoder.decodeArrayWithFactory(key, json: json)
}

func <~*~ <T: FactoryDecodable>(key: String, json: JSON) -> [String: T]? {
    return Decoder.decodeDictionaryWithFactory(key, json: json)
}
{% endhighlight %}

It would be nice to be able to use the `<~~` operator that Gloss uses,
but that gives us a "Type of expression is ambiguous without more context"
error. I'm not sure why that is, since `FactoryDecodable` is more specific
than `Decodable`.

Now we can have a `Decodable` class that contains arbitrary collections
of `Shape`s.

{% highlight swift %}
class ShapeHolder: Decodable {
    let shapes: [Shape]
    let byKey: [String: Shape]
    
    init?(json: JSON) {
        shapes = "shapes" <~*~ json ?? []
        byKey = "byKey" <~*~ json ?? [:]
    }
}
{% endhighlight %}

[last]: {{page.previous.url}} 
[gloss]: https://github.com/hkellaway/Gloss


