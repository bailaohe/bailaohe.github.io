---
layout: post
title: "Scala features to best practices [2]: companion object"
date: 2016-02-23
category: programming
tags: [scala, companion]
---

## FT-3: companion object

Define nonstatic (instance) members in your class, and define members that you want to appear as “static” members in an object that has the same name as the class, and is in the same file as the class. This object is known as a `companion object`.

```scala
// Pizza class
class Pizza (var crustType: String) {
  override def toString = "Crust type is " + crustType
}

// companion object
object Pizza {
  val CRUST_TYPE_THIN = "thin"
  val CRUST_TYPE_THICK = "thick"
  def getFoo = "Foo"
}
```

Although this approach is different than Java, the recipe is straightforward:

* Define your class and object in the same file, giving them the same name.
* Define members that should appear to be “static” in the object.
* Define nonstatic (instance) members in the class.

### SC-3-1: accessing private members

It’s also important to know that a class and its companion object can access each other’s private members. In the following code, the “static” method double in the object can access the private variable secret of the class Foo:

```scala
class Foo {
  private val secret = 2
}
object Foo {
  // access the private class field 'secret'
  def double(foo: Foo) = foo.secret * 2
}
object Driver extends App {
  val f = new Foo println(Foo.double(f)) // prints 4
}
```

Similarly, in the following code, the instance member printObj can access the private field obj of the object Foo:

```scala
class Foo {
  // access the private object field 'obj'
  def printObj { println(s"I can see ${Foo.obj}") }
}
object Foo {
  private val obj = "Foo's object"
}
object Driver extends App {
  val f = new Foo
  f.printObj
}
```

### SC-3-2: private primary constructor

A simple way to enforce the Singleton pattern in Scala is to make the primary constructor private, then put a getInstance method in the companion object of the class:

```scala
class Brain private {
  override def toString = "This is the brain."
}
object Brain {
  val brain = new Brain def getInstance = brain
}
object SingletonTest extends App {
  // this won't compile
  // val brain = new Brain

  // this works
  val brain = Brain.getInstance
      println(brain)
  }
}
```

### SC-3-3: creating instances without `new` keyword

```scala
class Person {
  var name: String = _
}
object Person {
  def apply(name: String): Person = {
    var p = new Person p.name = name
    p
  }
}
```

The `apply` method in a companion object is treated specially by the Scala compiler and lets you create new instances of your class without requiring the new keyword.

The problem can also be addressed by declaring your class as a `case class`. This works because the case class generates an apply method in a companion object for you. However, it’s important to know that a case class creates much more code for you than just the apply method. 
