---
layout: post
title:  Scala中的implicit
date:   2018-04-19 01:05:30 -0700
categories: Scala
---
最近在公司里和自己业余的时候重新上手Scala，在这里记录下来自己在写的时候学到的东西。之前虽然有Chris指路，但是我只是浅尝则止罢了。希望今次可以走得远一点，学得深一点。

回到本篇的主题，Scala的implicit功能很强大，好的implicit可以让代码更优雅，通过方法注入的方式实现类似于C#扩展方法的功能。

Scala编译器是按照怎样的规则来寻找一个可以应用的隐式转换方法呢？在"Programming in Scala"中是这么描述的：

> Implicit conversions are governed by the following general rules:
    1. Marking Rule: 只会去寻找带有implicit标记的方法。
    2. Scope Rule: 在一个作用域内，一个隐式转换必须是一个唯一的标识。一般的惯例，会将隐式转换方法封装在伴生对象中。
    3. Non-Ambiguity Rule: 只能有一个相应的隐式转换方法。
    4. One-at-a-time Rule: 只进行一次隐式转换。
    5. Explicits-First Rule: 显示优先。如果现有代码已经满足type check，则不会进行隐式转换。


举例来说，现在在我写的简单Scala代码中，我想通过Jsoup库拿到一个URL所对应的页面，即：

```scala
"http://www.google.com".fetch
```

然而，很明显String类并没有fetch方法。
我们先要提供一个Fetch的trait(定义typeclass的行为)

```scala
trait Fetch[-A, +B] {
  def fetch(source: A): B
}
object Fetch {
  def create[A, B](f: A => B): Fetch[A, B] = (source: A) => f(source)
}
```

为了方便使用，我们在Fetch typeclass里定义了构造函数。
下一步是注入fetch方法：

```scala
class FetchOps[-A, +B](a: A)(implicit ev: Fetch[A, B]) {
  def fetch: B = ev.fetch(a)
}
```

然后加上implicit resolution:

```scala
object FetchOps {
  implicit def toFetchOps[A, B](a: A)(implicit ev: Fetch[A,B]): FetchOps[A, B] = new FetchOps[A,B](a)
}
```

最后再定义一个String上的默认隐式转换，就完成了。用ScalaTest来写个test case

```scala
class FetchSpec extends FlatSpec with Matchers {
    "A url query" should "return a Document" in {
        import FetchOps._
        implicit val urlQuery: Fetch[String, Document] = Fetch.create(url => Jsoup.connect(url).get())
        "https://www.google.com".fetch shouldBe a [Document]
    }
}
```

