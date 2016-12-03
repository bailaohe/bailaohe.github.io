---
layout: post
title: "scala features to best practices [5]: implicits"
date: 2016-02-24 15:00:00
category: programming
tags: [scala, implicit]
---

Implicit conversions and implicit parameters are Scala’s power tools that do useful work behind the scenes. With implicits, you can provide elegant libraries that hide tedious details from library users.

## FT-6: implicit conversion (via implicit method/class)

An implicit conversion from type S to type T is defined by an implicit value which has function type S => T, or by an implicit method convertible to a value of that type. Implicit conversions are applied in two situations:

* If an expression e is of type S, and S does not conform to the expression’s expected type T.
* In a selection e.m with e of type T, if the selector m does not denote a member of T.

```scala
implicit def double2Int(d: Double) = d.toInt
val x: Int = 42.0
```

### SC-6-1: enrich an existing class

Rather than create a separate library of String utility methods, like a StringUtilities class, you want to add your own behavior(s) to the String class, so you can write code like this:

```scala
"HAL".increment
```
Instead of this:
```scala
StringUtilities.increment("HAL")
```

Then we can enrich the String class with an implicit method as follows:

```scala
// define a method named increment in a normal Scala class:
class StringImprovements(val s: String) {
  def increment = s.map(c => (c + 1).toChar)
}
// define another method to handle the implicit conversion:
implicit def stringToString(s: String) = new StringImprovements(s)
```

When you call `increment` on a String, which does not has that method at all. Thus, the compiler find the compatible one `StringImprovements` and convert the string to StringImprovements via the implicit method `stringToString`, this is the scenario-2 mentioned above.

Scala 2.10 introduced a new feature called implicit classes. An implicit class is a class marked with the implicit keyword. This keyword makes the class’ primary constructor available for implicit conversions when the class is in scope. This is similar to `monkey patching` in Ruby, and `Meta-Programming` in Groovy.

```scala
implicit class StringImprovements(s: String) {
  def increment = s.map(c => (c + 1).toChar)
}
```

In real-world code, this is just slightly more complicated. According to [SIP-13, Implicit Classes](http://docs.scala-lang.org/sips/completed/implicit-classes.html)

>An implicit class must be defined in a scope where method definitions are allowed (not at the top level).

This means that your implicit class must be defined inside a class, object, or package object. You can also check some other restrictions of implicit class here: [http://docs.scala-lang.org/overviews/core/implicit-classes.html](http://docs.scala-lang.org/overviews/core/implicit-classes.html)

## FT-7: implicit parameter

A method with implicit parameters can be applied to arguments just like a normal method. In this case the implicit label has no effect. However, if such a method misses arguments for its implicit parameters, such arguments will be automatically provided.

The actual arguments that are eligible to be passed to an implicit parameter fall into two categories:

* First, eligible are all identifiers x that can be accessed at the point of the method call without a prefix and that denote an implicit definition or an implicit parameter.
* Second, eligible are also all members of companion modules of the implicit parameter’s type that are labeled implicit.

### SC-7-1: default parameter value

Implicits can be used to declare a value to be provided as a default as long as an implicit value is set with in the scope.

```scala
def howMuchCanIMake_?(hours: Int)(implicit dollarsPerHour: BigDecimal) = dollarsPerHour * hours

implicit var hourlyRate = BigDecimal(34.00)
```

What's the advantage this solution takes over the simple default value in parameter definition? The search of implicit value can be taken in the scope of `companion object`, and thus you can keep the default value `private` from the caller.

### SC-7-2: implicit conversion via implicit parameter

An implicit function parameter is also usable as an implicit conversion, and it's more flexible than the traditional solution. Check the following codes:

```scala
def smaller[T](a: T, b: T)(implicit order: T => Ordered[T])
  = if (a < b) a else b // Calls order(a) < b if a doesn't have a < operator
```

Note that `order` is a function with a single parameter, is tagged implicit, and has a name that is a single identifier. Therefore, it is an implicit conversion, in addition to being an implicit parameter. So, we can omit the call to order in the body of the function
