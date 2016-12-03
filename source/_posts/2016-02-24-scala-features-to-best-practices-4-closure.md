---
layout: post
title: "scala features to best practices [4]: closure"
date: 2016-02-24 11:00:00
category: programming
tags: [scala, closure]
---

## FT-5: closure

You want to pass a function around like a variable, and while doing so, you want that function to be able to refer to one or more fields that were in the same scope as the function when it was declared.

In his excellent article, Closures in Ruby, Paul Cantrell states

>A closure is a block of code which meets three criteria

He defines the criteria as follows:

1. The block of code can be passed around as a value, and
2. It can be executed on demand by anyone who has that value, at which time
3. It can refer to variables from the context in which it was created (i.e., it is closed with respect to variable access, in the mathematical sense of the word “closed”).

Scala Cookbook, give a more graphic metaphor:

>I like to think of a closure as being like `quantum entanglement`, which Ein‐ stein referred to as “a spooky action at a distance.” Just as quantum entanglement begins with two elements that are together and then separated—but somehow remain aware of each other—a closure begins with a function and a variable defined in the same scope, which are then separated from each other. When the function is executed at some other point in space (scope) and time, it is magically still aware of the variable it referenced in their earlier time together, and even picks up any changes to that variable.

```scala
var votingAge = 18
val isOfVotingAge = (age: Int) => age >= votingAge

isOfVotingAge(16) // false
isOfVotingAge(20) // true

// change votingAge in one scope
votingAge = 21
// the change to votingAge affects the result
printResult(isOfVotingAge, 20) // now false

// `printResult` and `votingAge` can be far from each other in a light year
```
