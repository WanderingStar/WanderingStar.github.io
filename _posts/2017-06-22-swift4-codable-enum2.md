---
layout: post
title:  "Swift 4 Codable Enum with Associated Values 2"
date:   2017-06-22 21:51:00
tags: swift json codable unum
summary: >
  An example of a Codable enum without a subtype

---

In the [previous post][last], I gave an example of Coding/Decoding an `enum` with
associated values using a helper `struct` that mirrored the actual contents of the
JSON being parsed.

Here's an example of doing the same thing, but directly impementing
`init(from decoder:)` and `encode(to encoder:)`, instead of using the helper struct.

Seems cleaner, right? The only snag is that when I've tried the two approaches with
my real-world data, I've found that this direct approach is noticeably slower.
When decoding an array of 1000 `enums`, the helper struct version takes about 0.630s
and the direct decoding version takes about 0.733s. The effect seems to scale fairly
linearly (10000 takes 6.3s/7.3s etc.). Where's that extra 15% of the time going?
I don't know.

You can download a [playground here](/uploads/CodableEnum2.playground.zip).

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

[last]: {{page.previous.url}}
