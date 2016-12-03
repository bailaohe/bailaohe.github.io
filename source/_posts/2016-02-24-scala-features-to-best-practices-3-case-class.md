---
layout: post
title: "scala features to best practices [3]: case class"
date: 2016-02-24 10:00:00
category: programming
tags: [scala, case, match]
---

## FT-4: case class

### SC-4-1: build boilerplate code

You’re working with match expressions, actors, or other situations where you want to use the case class syntax to generate boilerplate code, including accessor and mutator methods, along with apply, unapply, toString, equals, and hashCode methods, and more.

Define your class as a case class, defining any parameters it needs in its constructor

```scala
// name and relation are 'val' by default
case class Person(name: String, relation: String)
```

Defining a class as a case class results in a lot of boilerplate code being generated, with the following benefits:

* An apply method is generated, so you don’t need to use the new keyword to create a new instance of the class.
* Accessor methods are generated for the constructor parameters because case class constructor parameters are val by default. Mutator methods are also generated for parameters declared as var.
* A good, default toString method is generated.
* An unapply method is generated, making it easy to use case classes in match expressions.
* equals and hashCode methods are generated.
* A copy method is generated.

### SC-4-2: pattern match via constructor pattern

```scala
trait Animal

case class Dog(name: String) extends Animal
case class Cat(name: String) extends Animal
case object Woodpecker extends Animal

object CaseClassTest extends App {
  def determineType(x: Animal): String = x match {
    case Dog(moniker) => "Got a Dog, name = " + moniker
    case _:Cat => "Got a Cat (ignoring the name)"
    case Woodpecker => "That was a Woodpecker"
    case _ => "That was something else"
}
```
