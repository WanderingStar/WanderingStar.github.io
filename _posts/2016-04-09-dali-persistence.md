---
layout: post
title:  "Dali: Persistence"
date:   2016-04-09 21:07:00
tags: swift gloss json datastore dali
summary: >
  A first cut at a Swift data store

---

After writing that [last post][last], I fell into the trap that
I was afraid I would: I spent my free coding time this week working on a
persistence scheme instead of on feature work on my app.

But I'm moderately happy with what I've come up with. As planned, it uses a weak
cache in front of [CouchBaseLite][cbl]. CBL is a document data store. It stores
indexed JSON, which we turn into Swift objects through [Gloss][gloss]-inspired
encoding and decoding.

The whole [thing][code] is under 100 lines of code so far, and a lot of that is
exception handling, so I think the ideas are pretty straightforward. It could
probably use a large helping of syntactic sugar and some thought put into
patterns for searching, but it's a start...

I've cheekily nicknamed it [dali][dali], after Salvador Dalí, in honor of
[The Persistence of Memory][art].

Here's what some familiar sample objects look like:

{% highlight swift %}
class Shape : Persistable {
    class var kind: String { return "Shape" }
    let identifier = NSUUID().UUIDString

    var area: Double {
        return 0.0
    }

    init() { }

    required init?(with json: JSON, from persistence: Persistence) throws { }

    func save(to: Persistence) throws {
        try to.save(self, json: [:])
    }
}

class Square : Shape {
    override class var kind: String { return "Square" }

    let side: Double

    override var area: Double {
        return side * side
    }

    init(side: Double) {
        self.side = side
        super.init()
    }

    required init?(with json: JSON, from persistence: Persistence) throws {
        guard let side: Double = "side" <~~ json else { return nil }
        self.side = side
        try super.init(with: json, from: persistence)
    }

    override func save(to: Persistence) throws {
        try to.save(self, json: ["side": side])
    }
}

class Circle : Shape {
    override class var kind: String { return "Circle" }

    let radius: Double

    override var area: Double {
        return M_PI * radius * radius
    }

    init(radius: Double) {
        self.radius = radius
        super.init()
    }

    required init?(with json: JSON, from persistence: Persistence) throws {
        guard let radius: Double = "radius" <~~ json else { return nil }
        self.radius = radius
        try super.init(with: json, from: persistence)

    }

    override func save(to: Persistence) throws {
        try to.save(self, json: ["radius": radius])
    }
}
{% endhighlight %}

Relationships are simply stored identifiers.

{% highlight swift %}

final class VennDiagram : Persistable {
    static let kind = "VennDiagram"
    let identifier = NSUUID().UUIDString

    let left: Circle
    let right: Circle

    init(left: Circle, right: Circle) {
        self.left = left
        self.right = right
    }

    required init?(with json: JSON, from persistence: Persistence) throws {
        guard let left: Circle = try persistence.load("left" <~~ json),
            right: Circle = try persistence.load("right" <~~ json)
            else { return nil }
        self.left = left
        self.right = right
    }

    func save(to: Persistence) throws {
        try left.save(to)
        try right.save(to)
        try to.save(self, json: ["left": left.identifier, "right": right.identifier])
    }

}
{% endhighlight %}

They can be lazily instantiated if desired.

{% highlight swift %}
final class LazySquare : Persistable {
    static let kind = "LazySquare"
    let identifier = NSUUID().UUIDString

    private weak var persistence: Persistence?
    private var squareIdentifier: String?
    lazy var square: Square? = try! self.persistence?.load(self.squareIdentifier)

    init(square: Square) {
        self.square = square
    }

    required init?(with json: JSON, from persistence: Persistence) throws {
        self.persistence = persistence
        self.squareIdentifier = "square" <~~ json
    }

    func save(to: Persistence) throws {
        if let square = square {
            try square.save(to)
            try to.save(self, json: ["square": square.identifier])
        } else {
            try to.save(self, json: [:])
        }
    }
}
{% endhighlight %}

[last]: {{page.previous.url}}
[gloss]: https://github.com/hkellaway/Gloss
[cbl]: http://developer.couchbase.com/documentation/mobile/current/get-started/couchbase-lite-overview/index.html
[code]: https://github.com/WanderingStar/dali/blob/7be2da80c4249853d7f1a5ad50bc38145dcb44a6/Dali/Persistence.swift
[dali]: https://github.com/WanderingStar/dali
[art]: http://www.wikiart.org/en/salvador-dali/the-persistence-of-memory-1931
