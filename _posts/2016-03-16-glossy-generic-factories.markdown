---
layout: post
title:  "Glossy Generic Factories"
date:   2016-03-16 23:46:29 -0500
---
[Last time][last] we looked at creating one-off factories that we can use to 
parse collections of subclass objects with [Gloss][gloss].

This time we're going to take a step back and use generics to abstract out
the Factory...

We'll start off with a protocol that looks a lot like
Decodable. There's probably a way to use Decodable, but I couldn't get
the operators working properly.

{% highlight swift %}
protocol Makeable {
    init?(json: JSON)
}
{% endhighlight %}

Here is our Factory. It knows the keyPath to look at to find the
type identifier in a JSON block, and it has a table of subtypes to
create based on what it finds there.

{% highlight swift %}
class Factory<T: Makeable> {
    let keyPath: String
    var subtypes = [String: T.Type]()
    
    init(typeKeyPath: String) {
        keyPath = typeKeyPath
    }
    
    func make(json: JSON) -> T? {
        if let type = json.valueForKeyPath(keyPath) as? String,
            subtype = subtypes[type] {
                return subtype.init(json: json)
        }
        return nil
    }
}
{% endhighlight %}

Our subtypes should implement this protocol, specifying what
factory to use to decode them.

{% highlight swift %}
protocol FactoryDecodable: Makeable {
    typealias Output: Makeable
    static var factory: Factory<Output> { get }
}
{% endhighlight %}

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

There's nothing special in our subclasses...

{% highlight swift %}
class Square: Shape {
    let side: Double
    override var area: Double { return side * side }
    
    required init?(json: JSON) {
        if let side = json["side"] as? Double {
            self.side = side
            super.init(json: json)
        } else {
            self.side = 0           // ugh
            super.init(json: json)  // ugh
            return nil
        }
    }
}


class Circle: Shape {
    let radius: Double
    override var area: Double { return M_PI * radius * radius }
    
    required init?(json: JSON) {
        if let radius = json["radius"] as? Double {
            self.radius = radius
            super.init(json: json)
        } else {
            self.radius = 0         // ugh
            super.init(json: json)  // ugh
            return nil
        }
    }
}
{% endhighlight %}

Our subtypes must be registered with the SuperType's Factory.

{% highlight swift %}
Shape.factory.subtypes["square"] = Square.self
Shape.factory.subtypes["circle"] = Circle.self
{% endhighlight %}

And now the Glossy part... Decoders that look for FactoryDecodables
and collections of them.

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

And operators to make those nicer to use. 

{% highlight swift %}
func <~~ <T: FactoryDecodable>(key: String, json: JSON) -> T? {
    return Decoder.decodeWithFactory(key, json: json)
}

func <~~ <T: FactoryDecodable>(key: String, json: JSON) -> [T]? {
    return Decoder.decodeArrayWithFactory(key, json: json)
}

func <~~ <T: FactoryDecodable>(key: String, json: JSON) -> [String: T]? {
    return Decoder.decodeDictionaryWithFactory(key, json: json)
}
{% endhighlight %}

Now we can have a Decodable class that contains arbitrary collections
of Shapes.

{% highlight swift %}
class ShapeHolder: Decodable {
    let shapes: [Shape]
    let byKey: [String: Shape]
    
    init?(json: JSON) {
        shapes = "shapes" <~~ json ?? []
        byKey = "byKey" <~~ json ?? [:]
    }
}
{% endhighlight %}

[last]: {{page.previous.url}} 
[gloss]: https://github.com/hkellaway/Gloss

