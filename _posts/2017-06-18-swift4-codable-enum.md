---
layout: post
title:  "Swift 4 Codable Enum with Associated Values"
date:   2017-06-18 21:51:00
tags: swift json codable unum
summary: >
  An example of a Codable enum

---

Well, it's been more than a year since I last posted here... I ran into the limitations
of the reflection system and then got distracted by other things.

Just a quick post to provide an example of a `Codable` `enum` with associated values
in Swift 4, since I couldn't find one with a trivial search...

This `enum` has two cases, each with associated values (of `Codable` types). We represent
it in JSON as an `object` (which is like a Swift dictionary). Because there are dictionaries
that are well-formed JSON, but don't represent correct Shapes, we have to check the
assumptions as we Decode, and throw errors.

Of course, this is a toy example. In a real-world situation, you'd want to chose a JSON
representation that more closely matches your structure. Having a single key that corresponds
to each `enum` case would probably make sense.

You can download a [playground here](/uploads/CodableEnum.playground.zip).

{% highlight swift %}
import Foundation

enum Shape: Codable {
    case rectangle(Double, Double)
    case circle(Double)
    
    enum CodingKeys: String, CodingKey {
        case length
        case width
        case radius
    }
    
    // These are the ways the struct could be well-formed JSON, but incoherent
    enum ShapeDecodingError: Error {
        case missingLength
        case missingWidth
        case squaringTheCircle
        case shapeless
    }
    
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        let length: Double? = try? container.decode(Double.self, forKey: .length)
        let width: Double? = try? container.decode(Double.self, forKey: .width)
        let radius: Double? = try? container.decode(Double.self, forKey: .radius)
        
        switch (length, width, radius) {
        case (nil, nil, nil):
            throw ShapeDecodingError.shapeless
        case (nil, _, nil):
            throw ShapeDecodingError.missingLength
        case (_, nil, nil):
            throw ShapeDecodingError.missingWidth
        case (.some(let l), .some(let w), nil):
            self = .rectangle(l, w)
        case (nil, nil, .some(let r)):
            self = .circle(r)
        default:
            throw ShapeDecodingError.squaringTheCircle
        }
    }
    
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        switch self {
        case .rectangle(let length, let width):
            try container.encode(length, forKey: .length)
            try container.encode(width, forKey: .width)
        case .circle(let radius):
            try container.encode(radius, forKey: .radius)
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
