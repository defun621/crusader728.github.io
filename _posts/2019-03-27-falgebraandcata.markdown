---
layout: post
title:  F algebra 与 catamorphisms
date:   2019-03-27 00:09:30 -0700
categories: 
    - Programming
    - Scala 
---
书接上文，有了Fixed data type之后，我们原来的mkString 和 count 该怎么改写？
```scala
def mkString(e: Fix[EmployeeF]): String = ???

def count(e: Fix[EmployeeF]): Int = ???
```

我们先看看我们手头有什么？ 

# EmployeeF\[A\]

假设我们已经有了EmployeeF\[String\]，则eval出最后的String是很简单的

```scala
def eval(e: EmployeeF[String]): String = e match {
  case Engineer(n) => n
  case Manager(n, l) => s"$n:[${l.mkString(",")}]"
}
```

同理，假设我们已经有了EmployeeF\[Int\]，则eval出最后的员工人数也很简单：

```scala
def count(e: EmployeeF[Int]): Int = e match {
  case Engineer(_) => 1
  case Manager(_, l) => 1 + l.sum
}
```

我们可以从中找到共通点：

```scala
type Algebra[F[_], A] = Function[F[A],A]

type CountA = Algebra[EmployeeF, Int]
val countA: CountA = count

type MkStringA = Algebra[EmployeeF, String]
val mkStringA: MkStringA = eval
```

其实上面，我们就已经差不多构造了一个EmployeeF上的F-Algebra。

一个F-algebra由以下三点组成：

1. 一个自函子F；
2. 一个范畴中的对象A；
3. 一个F(A)到A的态射。

# 自函子F

F需要是自函子，我是这么理解的: 我们可以从参数 Fix\[EmployeeF\]来出发。

我们对于参数Fix\[EmployeeF\]能做的事情很有限，就是只有拆包：

```scala
def unfix[F[_]](x: Fix[F]): F[Fix[F]] = x match {
  case Fix(a) => a
}
```

所以，如果我们的EmployeeF是个自函子，那么如果有函数 g 能从 Fix\[EmployeeF\] 映射到 A，那我们就可以通过map g 从EmployeeF\[Fix\[EmployeeF\]\] 映射到 EmployeeF\[A\]。

# Catamorphism

所以整条路应该是 unfix --> map g --> alg
```haskell
g = alg . (fmap g) . unFix
```
使得g = cata alg，而cata就是如下的函数

用scala来写就是
```scala
import scalaz.Functor
import scalaz.syntax.functor._

def cata[F[_]: Functor, A](alg: F[A] => A): Fix[F] => A = f => alg.apply(f.unfix fmap cata(alg))
```

# 该怎么写mkString和count？
有了catamorphism和alg, 写mkString和count就很直接了：
![worksheet result]({{ site.url }}/assets/catamorphism.png)

# References

1. [Understanding F-Algebras by Bartosz Milewski](https://www.schoolofhaskell.com/user/bartosz/understanding-algebras)
2. [Peeling the Banana: Recursion Schemes from First Principles - Zainab Ali](https://www.youtube.com/watch?v=XZ9nPZbaYfE)
3. [F-algebra by 夏梓耀](https://zhuanlan.zhihu.com/p/21354189)
