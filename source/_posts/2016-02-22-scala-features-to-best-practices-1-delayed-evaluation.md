---
layout: post
title: "Scala features to best practices [1]: delayed evaluation"
date: 2016-02-22
category: programming
tags: [scala, delayed, by-name, lazy]
---

## features: delayed evaluation

I'd like to use the term `delayed evaluation` to cover following two features in scala: `lazy var/val` and `byname parameter`. They are not quite related to each other, but both are to postpone the evaluation of a given expression or block for the final result.

## FT-1: lazy var/val

Defining a field as `lazy` is a useful approach when the field might not be accessed in the normal processing of your algorithms, or if running the algorithm will take a long time, and you want to defer that to a later time.

At present, I think its useful in following scenarios:

### SC-1-1: field-initialization takes great efforts

It makes sense to use `lazy` on a class field if its initialization requires a long time to run, and we don't want to do the job when we instantiate the class until we actually use the field.

```scala
class Foo {
  lazy val text = io.Source.fromFile("/etc/passwd").getLines.foreach(println)
}

object Test extends App {
  val f = new Foo
}
```

In above example, the initialization of `text` needs to retrieve the contents of the text file `/etc/passwd`. But when this code is compiled and run, there is no output, because the text field isn’t initialized until it’s accessed. That’s how a lazy field works.

### SC-1-2: field-initialization has dependencies

Sometimes we need to initialize fields in a specific order because they have dependency on the other. Then we may produce following ugly codes:

```scala
class SparkStreamDemo extends Serializable {

  @transient private var conf: SparkConf = null
  @transient private var sc: SparkContext = null
  @transient private var ssc: StreamingContext = null

  def getConf() = {
    if (conf == null)
      conf = new SparkConf()
    conf
  }

  def getSC() = {
    if (sc == null)
      sc = new SparkContext(getConf)
    sc
  }

  def getSSC() = {
    if (ssc == null)
      ssc = new StreamingContext(getSC, Seconds(10))
    ssc
  }
}
```

In this spark-streaming demo, the initialization of `ssc` depends on that of `sc`, which further depends on `conf`. We operate the initialize manually, thus we define these fields with `var`, and implement the lazy initialization in getters. The shortcoming is obvious, we have to restrict the access of these field through getters, otherwise we may get the null-valued ones! Moreover, defining `var` to these fields is not best-practice since they are readonly after initialization. A modified version via `lazy val/var` is as follows:

```scala
class SparkStreamDemo extends Serializable {
  @transient lazy val conf: SparkConf = new SparkConf()
  @transient lazy val sc: SparkContext = new SparkContext(conf)
  @transient lazy val ssc: StreamingContext = new StreamingContext(sc, Seconds(10))
}
```

What if a `lazy` field depends on a `non-lazy` var, which is not properly initialzed? Can the instance be re-used after some `NullPointerException`-like error raised? This seems no problem as scala provides a tricky, as @ViktorKlang posted on Twitter:

>Little known Scala fact: if the initialization of a lazy val throws an exception, it will attempt to reinitialize the val at next access.

You can check the details here: [http://scalapuzzlers.com/#pzzlr-012](http://scalapuzzlers.com/#pzzlr-012)

## FT-2: by-name parameter

The by-name parameter can be considered equivalent to `() => Int`, which is a `Function type` that takes a Unit type argument. Besides from normal functions, it can also be used with an `Object` and `apply` to make interesting block-like calls.

### SC-2-1: wrapper function

```scala
// A benchmark construct:
def benchmark (body : => Unit) : Long = {
  val start = java.util.Calendar.getInstance().getTimeInMillis()
  body
  val end = java.util.Calendar.getInstance().getTimeInMillis()
  end - start
}

val time = benchmark {
  var i = 0 ;
  while (i < 1000000) {
    i += 1 ;
  }
}

println("while took:   " + time)
```

### SC-2-2: Add syntactic sugar

```scala
// While loops are syntactic sugar in Scala:
def myWhile (cond : => Boolean) (body : => Unit) : Unit =
  if (cond) { body ; myWhile (cond) (body) } else ()

var i = 0 ;

myWhile (i < 4) { i += 1 ; println (i) }
```

Accompany with `curry`, we re-implement a while-loop in above example.
