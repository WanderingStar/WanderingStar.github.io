---
layout: post
title:  "Swift 4 Codable Enum"
date:   2017-06-18 21:51:00
tags: swift json codable unum
summary: >
  An example of a Codable enum

---

Well, it's been more than a year since I last posted here... I ran into the limitations
of the reflection system and then got distracted by other things.

Just a quick post to provide an example of a `Codable` `enum` in Swift 4, since I couldn't
find one with a trivial search...

This `enum` has two cases, each with associated values (of `Codable` types). We represent
it in JSON as an `object` (which is like a Swift dictionary). Because there are dictionaries
that are well-formed JSON, but don't represent correct Shapes, we have to check the
assumptions as we Decode, and throw errors.

You can download a [playground here](/uploads/CodableEnum.playground.zip).

{% highlight swift %}
import Foundation

enum Shape: Codable {
    case rectangle(Double, Double)
    case circle(Double)
    
    // the enum is not inherently Codable
    init(from decoder: Decoder) throws {
        let shapeDictionary = try ShapeDictionary(from: decoder)
        self = try Shape.init(shapeDictionary: shapeDictionary)
    }
    
    func encode(to encoder: Encoder) throws {
        try ShapeDictionary(shape: self).encode(to: encoder)
    }
    
    // This struct _is_ Codable, though ambiguous
    struct ShapeDictionary: Codable {
        let length: Double?
        let width: Double?
        let radius: Double?
        
        init(shape: Shape) {
            switch shape {
            case .rectangle(let l, let w):
                self.length = l
                self.width = w
                self.radius = nil
            case .circle(let r):
                self.length = nil
                self.width = nil
                self.radius = r
            }
        }
    }
    
    // These are the ways the struct could be well-formed JSON, but incoherent
    enum ShapeDictionaryDecodingError: Error {
        case missingLength
        case missingWidth
        case squaringTheCircle
        case shapeless
    }
    
    init(shapeDictionary: ShapeDictionary) throws {
        switch (shapeDictionary.length, shapeDictionary.width, shapeDictionary.radius) {
        case (nil, nil, nil):
            throw ShapeDictionaryDecodingError.shapeless
        case (nil, _, nil):
            throw ShapeDictionaryDecodingError.missingLength
        case (_, nil, nil):
            throw ShapeDictionaryDecodingError.missingWidth
        case (.some(let l), .some(let w), nil):
            self = .rectangle(l, w)
        case (nil, nil, .some(let r)):
            self = .circle(r)
        default:
            throw ShapeDictionaryDecodingError.squaringTheCircle
        }
    }
}

let shapes = [Shape.rectangle(1.0, 2.0), Shape.circle(3.0)]

let encoder = JSONEncoder()
let jsonString = try String(data: encoder.encode(shapes), encoding:.utf8)

let decoder = JSONDecoder()
let json = """
[
    {
        "radius": 4.0
    },
    {
        "length": 5.0,
        "width": 6.0
    }
]
""".data(using: .utf8)!

let shapes2 = try! decoder.decode([Shape].self, from: json)
shapes2[0]
shapes2[1]
{% endhighlight %}
