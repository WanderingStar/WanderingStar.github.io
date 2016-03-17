---
layout: post
title:  "Glossy Factories"
date:   2016-03-16 21:40:29 -0500
categories: swift glossy json
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
an arbitrary order.

I think Gloss lends itself to a pretty nice Factory pattern solution
to this problem...

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
Shape.subtypes["square"] = Square.self

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
Shape.subtypes["circle"] = Circle.self

if let c = Shape.factory(["type": "circle", "radius": 3.0]) {
    print(c.area)
}
if let s = Shape.factory(["type": "square", "side": 6.0]) {
    print(s.area)
}
print(Shape.factory(["type": "unknown", "xyzzy": 11]) == nil)
print(Shape.factory(["type": "circle"]) == nil)


/* using Gloss... */

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


/* and elsewhere ... */


class ShapeHolder {
    let shapes: [Shape]
    let byKey: [String: Shape]
    
    init?(json: JSON) {
        self.shapes = "shapes" <~~ json ?? []
        self.byKey = "byKey" <~~ json ?? [:]
    }
}


let filePath = NSBundle.mainBundle().pathForResource("shapes", ofType:"json")
let data = try! NSData(contentsOfFile:filePath!,
    options: NSDataReadingOptions.DataReadingUncached)
let json = try! NSJSONSerialization.JSONObjectWithData(data,
    options: NSJSONReadingOptions()) as! JSON
if let holder = ShapeHolder(json: json) {
    print("array: \(holder.shapes)")
    print("Dictionary: \(holder.byKey)")
    
    print("arrayArea: \(holder.shapes.reduce(0, combine: { $0 + $1.area })) = 9 + π")
    print("arrayArea: \(holder.byKey.values.reduce(0, combine: { $0 + $1.area })) = 25 + 100π")
}
{% endhighlight %}

But we can do better...
 
[gloss]: https://github.com/hkellaway/Gloss
[swifty]: https://github.com/SwiftyJSON/SwiftyJSON
[cbl]: http://developer.couchbase.com/documentation/mobile/current/get-started/couchbase-lite-overview/index.html

