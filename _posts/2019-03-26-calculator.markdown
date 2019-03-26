---
layout: post
title:  关于如何在on site的时候装逼之calculator
date:   2019-03-26 00:57:30 -0700
categories: 
    - Programming
    - Scala 
---
最近公司招人面试有点频繁，经常会有on site。然而我是一个很懒的人，懒得想太多面试题。

所以我常常会出的题就是让面试者写一个evaluate方法，对于给定的可能包含+, -, *, /以及开闭括号的字符串计算出它的值。

面试者往往上来就开stack，然后for循环一把。虽然没错，但是都9012年了，怎么还没人手写combinator 来解呢？（手动摇扇

今天晚上睡不着，试着写一个combinator 玩玩。

# Parser
首先我们来定义一下Parser出来的结果
```scala
sealed abstract class Result[+T]

case object Fail extends Result[Nothing]

case class Succ[A](v: A, remaining: String) extends Result[A]
```

Parse可能失败，可能成功。

+ 成功时返回Succ，其中v是parse出来的结果，remaining是剩余的字符串。
+ 失败的时候，直接返回Fail

再接下来定义一下Parser。
```scala
trait Parser[A] { self =>
  def parse(input: String): Result[A]
}
```

再加上map方法和flatMap方法，我们的Parser就完成了（不提monad和functor）。
```scala
trait Parser[A] { self =>
  def parse(input: String): Result[A]

  def flatMap[B](f: A => Parser[B]): Parser[B] = (input: String) => self.parse(input) match {
    case Succ(v, remaining) => f(v).parse(remaining)
    case Fail => Fail
  }

  def map[B](f: A => B): Parser[B] = (input: String) => self.parse(input) match {
    case Succ(v, remaining) => Succ(f(v), remaining)
    case Fail => Fail
  }
}
```

# 组合子
定义完了接口，我们首先来定义一些基本的元素。pure和fail，其中pure方法返回的parser永远成功， 相对应的fail则永远失败。
```scala
def pure[A](a: A): Parser[A] = (input: String) => Succ[A](a, input)

def fail[A]: Parser[A] = _ => Fail
```

接下来是分支和循环
```scala
def or[A](left: Parser[A], right: Parser[A]): Parser[A] = (input: String) => left.parse(input) match {
  case s@Succ(_, _) => s
  case Fail => right.parse(input)
}
```
我们先尝试左边的parser。如果成功则直接返回，否则就用右边的parser来parse输入。

```scala
def many[A](p: Parser[A]): Parser[List[A]] = or(many1(p), pure(Nil))

def many1[A](p: Parser[A]): Parser[List[A]] = for {
  x <- p
  xs <- many(p)
} yield x::xs
```
两个函数各自调用彼此。many表示至少1个以上或者空。many1也很好理解。

# 组装
有了以上这些基本元素，我们就可以开始动手读输入字符串了。
首先先定义一个Parser，每次都读一个字符进来
```scala
val item: Parser[Char] = input => input.toList match {
  case Nil => Fail
  case x :: xs => Succ[Char](x, xs.mkString)
}
```

我们可能需要读进来的字符满足某些条件，比如说是个数字，是个特定字符（+，-，*,/,括号），所以得给item配一个predicate
```scala
def sat(predicate: Char => Boolean): Parser[Char] = item.flatMap(c => if(predicate(c)) pure(c) else fail)
```

这样的话我们就一下子把整个数字读进来了
```scala
val nat: Parser[Int] = many1(sat(c => Character.isDigit(c))).map(l => l.mkString.toInt)
```

也可以判定是否是我们关心的符号
```scala
def symbol(char: Char): Parser[Char] = sat(c => c.equals(char))
```

最后，根据语法，我们可以给出这样的递归定义：
```scala
//factor := (expr) | nat
def factor: Parser[Int] = or(for { 
  _ <- symbol('(')
  e <- expr
  _ <- symbol(')')
} yield e, nat)


//expr := term (+ expr | - expr | END)
def expr: Parser[Int] = {
  for {
    t <- term
    r <- or(for {
      op <- or(symbol('+'), symbol('-'))
      e <- expr
      result <- op match {
        case '+' =>
          pure(t + e)
        case '-' =>
          pure(t - e)
      }
    } yield result, pure(t))
  } yield r
}

// term := factor (* term | / term | END)
def term: Parser[Int] = {
  for {
    f <- factor
    r <- or(for {
      op <- or(symbol('*'), symbol('/'))
      t <- term
      result <- op match {
        case '*' =>
          pure(f * t)
        case '/' =>
          pure(f / t)
      }
    } yield result, pure(f))
  } yield r
}
```

创建一个worksheet试试看
![结果]({{ site.url }}/assets/calculatorworksheet.png)

成功！

# Reference
1. Hutton, Graham. Programming in Haskell. 2nd ed., Cambridge University Press, 2016.

