---
layout: post
title:  "Glossy Factories"
date:   2016-03-16 21:40:29 -0500
tags: swift gloss json
---
I've been playing around with [Gloss][gloss], a Swift JSON parser.
I'm liking the mechanics of it a bit better than [SwiftyJSON][swifty]
for a few reasons:

- More concise code
- Takes its arguments as `[String: AnyObject]`, which works nicely
  with the native types of NSJSONSerialization and [CouchBase Lite][cbl]

One of my perennial problems with JSON parsing in Swift is mixed
collections of objects. For example, maybe you have a collection of
Shapes. Some of them are Circles, some are Squares. They may be in
an arbitrary order. When you parse them out of the JSON, you want
them to be the proper subtypes.

I think Gloss lends itself to a pretty nice Factory pattern solution
to this problem...

Here we have our supertype. It has a factory method, and a mapping
from subtype name to Type.

{% highlight swift %}
typealias JSON = [String: AnyObject]

class Shape {
    static var subtypes = [String: Shape.Type]()
    
    required init?(json: JSON) {
        // base class does nothing
    }
    
    static func factory(json: JSON) -> Shape? {
        if let type = json["type"] as? String,
            subtype = Shape.subtypes[type] {
                return subtype.init(json: json)
        }
        return nil
    }
    
    var area: Double { return 0 }
}
{% endhighlight %}

There's nothing special about our subclasses.

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

The subclasses must be registered with the factory. It would be
nice to do this when the class is created, but Swift doesn't offer
us something like `+initialize`. This can go somewhere like the
AppDelegate's launch method.

{% highlight swift %}
Shape.subtypes["square"] = Square.self
Shape.subtypes["circle"] = Circle.self
{% endhighlight %}

So far I haven't mentioned Gloss. With Gloss, we can add decoder
methods and `<~~` convenience operators.

{% highlight swift %}
extension Decoder {
    
    static func decodeShape(key: String, json: JSON) -> Shape? {
        if let shapeJson = json.valueForKeyPath(key) as? JSON {
            return Shape.factory(shapeJson)
        }
        return nil
    }
    
    static func decodeShapeArray(key: String, json: JSON) -> [Shape]? {
        if let shapesJson = json.valueForKeyPath(key) as? [JSON] {
            return shapesJson.flatMap { shapeJson in Shape.factory(shapeJson) }
        }
        return nil
    }
    
    static func decodeShapeDictionary(key: String, json: JSON) -> [String: Shape]? {
        var shapeDictionary = [String: Shape]()
        if let shapesJson = json.valueForKeyPath(key) as? [String: JSON] {
            for (shapeKey, shapeJson) in shapesJson {
                shapeDictionary[shapeKey] = Shape.factory(shapeJson)
            }
            return shapeDictionary
        }
        return nil
    }
    
}

func <~~ (key: String, json: JSON) -> Shape? {
    return Decoder.decodeShape(key, json: json)
}

func <~~ (key: String, json: JSON) -> [Shape]? {
    return Decoder.decodeShapeArray(key, json: json)
}

func <~~ (key: String, json: JSON) -> [String: Shape]? {
    return Decoder.decodeShapeDictionary(key, json: json)
}
{% endhighlight %}

This lets us parse those varied collections as cleanly as:

{% highlight swift %}
class ShapeHolder {
    let shapes: [Shape]
    let byKey: [String: Shape]
    
    init?(json: JSON) {
        self.shapes = "shapes" <~~ json ?? []
        self.byKey = "byKey" <~~ json ?? [:]
    }
}
{% endhighlight %}

Pretty nice! But [we can do better...][next]
 
[gloss]: https://github.com/hkellaway/Gloss
[swifty]: https://github.com/SwiftyJSON/SwiftyJSON
[cbl]: http://developer.couchbase.com/documentation/mobile/current/get-started/couchbase-lite-overview/index.html
[next]: {{page.next.url}}
