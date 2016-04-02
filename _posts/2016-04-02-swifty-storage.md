---
layout: post
title:  "Swifty Storage"
date:   2016-04-02 12:30:00
tags: swift gloss json datastore realm
summary: >
  What would make a good Swift data store?

---

I've been working on a rewrite of [Barbell Builder](http://barbellbuilder.com)
in Swift. It was going really well until I ran into the limitations of my
object persistence scheme, and had trouble finding something that met all of
my desires.

This is me trying to organize my thoughts about that. I'm resisting the
temptation to try to build the system I want, because I know that that's the
kind of project that will consume my attention and distract me from finishing
the project I'm actually working on.

Desired features:

- Swifty
    - Proper treatment of optionals (and _non_-optionals)
    - let properties are immutable
    - Enums
- Polymorphism
- Collections
- Lazy fetching where appropriate
- Querying
- "Live" objects: a change to one instance updates it elsewhere

## Core Data
Cons:

- Un-Swifty
    - Objective-C style objects
    - no let properties, everything is var
    
Pros:

- Lazy fetching
- Well supported

## JSON Files
Cons:

- How to query?
    - By type/id is simple
    - By content requires building an index and keeping it up to date

Pros:

- Custom (Glossy) encode/decode allows support for arbitrary Swift types

## JSON in SQLite
Same as JSON Files, but easier to organize

## JSON in CouchBaseLite
Pros:

- Querying

## Realm
Cons:

- Un-Swifty
    - Objective-C style objects
    - Must use Classes
    - no let properties, everything is var
    - Bad treatment of optionals, enums, structs, polymorphism, maps

Pros:

- Live objects
- KVO
- Search/filter via NSPredicate

## Idea

Okay, so I know that I said that I was resisting the temptation, but what if...

- Weak cache ([NSMapTable](http://nshipster.com/nshashtable-and-nsmaptable/)?)
  in front of JSON-based data store (CouchBaseLite?)
    - "Live" objects because single instance
- lazy stored properties for relationships

Might be time to write some prototype code...
