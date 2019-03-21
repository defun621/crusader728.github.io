---
layout: post
title:  Fixpoint Types
date:   2019-03-20 19:39:30 -0700
categories: 
    - Programming
    - Scala 
---

为了看懂Free.scala源代码, 花了一点时间了解了一些必需的知识。接下来会分几篇来记录自己了解到的FP知识。今天先说的是Fixpoint types

# ADT Employee 作为例子

假如我们有如下recursive data type来表达员工层级： 

```scala
sealed abstract class Employee(name: String)

case class Manager(name: String, subordinates: List[Employee]) extends Employee(name)

case class Engineer(name: String) extends Employee(name)

// example: val li = Manager("Li", 
//                          Manager("Tommy", 
//                                  List(Engineer("Vince"), Engineer("Lin"), Engineer("Ian"))), 
//                          Manager("Chris",
//                                  List(Engineer("James"), Engineer("Henry"))))
```

对这样的ADT，我们一般都用pattern matching来解构。比如说我要写一个toString方法，我可以这么写

```scala
val mkString: Employee => String = {
  case Manager(name, subordinates) => name + ":[" + subordinates.map(e => mkString(e)).mkString(",") + "]"
  case Engineer(name) => name
}

//mkString(li) returns "Li:[Tommy:[Vince,Lin,Ian],Chris:[James,Henry]]"
```

又比如，我想统计部门里有多少人，可以这么写：

```scala
val count: Employee => Int = {
  case Manager(_, subordinates) => 1 + subordinates.map(e => count(e)).reduceLeft(_+_)
  case Engineer(_) => 1
}

//count(li) returns 8
```

# 如果我们要存DB的话？

一般每个数据存到DB都会有unique id与之相对应，所以我们需要改动一下我们的Employee数据结构，加入一个id。可能会像下面这样改写

```scala
sealed abstract class Employee(id: Option[Long], name: String)

case class Manager(id: Option[Long], name: String, subordinates: List[Employee]) extends Employee(id, name)

case class Engineer(id: Option[Long], name: String) extends Employee(id, name)
```

但是这样并不好，为什么？在写代码的时候，一共就如下几种情况：

1. 刚刚构造出这个新的data object，这时候没有id
2. data object 是从db load出来的，这时候有id
3. 我们不关心id，因为id和逻辑没关系，就像上面的count或mkString一样

把id这个另外一层逻辑（存储）的数据放在ADT里很显然增大了我们ADT的Complexity。所以单独拿出来会是更优的方案。于是我们得到了

```scala
sealed abstract class Employee(name: String)

case class Manager(name: String, subordinates: List[Employee]) extends Employee(name)

case class Engineer(name: String) extends Employee(name)

type Record = (Long, Employee)
```

看上去很美。但是问题来了，subordinates这里又没有了id的信息。然而我们如果把subordinates的类型改成List\[(Long, Employee)\]，之前的改动就没有了意义，因为我们就是要把Long从Employee里拿出去。

# 加个Type parameter？

遇到这种不知道放什么类型好的时候，youtube里的大神说要有type parameter！所以加个type parameter试试看？我们得到了：

```scala
sealed abstract class EmployeeF[A](name: String)

case class Manager[A](name: String, subordinates: List[A]) extends EmployeeF[A](name)

case class Engineer[A](name: String) extends EmployeeF[A](name)
```

问题又来了: _Engineer\[Unit\]("Ian")_ 是 __EmployeeF\[Unit\]__ 类型， 那 _List(Engineer\[Unit\]("ian"))_ 的类型就是 __List\[EmployeeF\[Unit\]\]__ ， 所以在Manager那里A就是 __Employee\[Unit\]__ 。这样的话在之前的例子里，Manager tommy 就变成了 __EmployeeF\[EmployeeF\[Unit\]\]__ ，Manager li 的类型就变成了 __EmployeeF\[EmployeeF\[EmployeeF\[Unit\]\]\]__ 。

这可如何是好？

# Fixpoint data type

这种递归的类型，在Scala里可以解。

> 递归是可以被抽象出来的，做法就是定义一个非递归的函数，并且找到它的不动点

在Scala里，我们用下面这个case class来表达不动点：
```scala
case class Fix[F[_]](unfix: F[Fix[F]])
```

有了Fix，我们可以这么定义我们的Employee
```scala
type Employee = Fix[EmployeeF]
```

无限递归的问题解决了！

我们来验证一下：
_Fix(Engineer\[Fix\[EmployeeF\]\])("Ian")_ 的类型是 __Fix\[EmployeeF\]__ ，所以 _List(Fix(Engineer\[Fix\[EmployeeF\]\]("Ian")))_ 的类型是 __List\[Fix\[EmployeeF\]\]__ 。所以在Manager那里A就是 __Fix\[EmployeeF\]__ 。这样的话在之前的例子里，Manager tommy 就还是 __Fix\[EmployeeF\]__ ，Manager li 的类型同样也还是 __Fix\[EmployeeF\]__ 。

# 绕了这么大一圈，就算是抽象出了不动点，我们原来的mkString 和 count 该怎么改写？
等我看懂了catamorphism我再来写一下(/▽＼)

# Reference

1. [Pure Functional Database Programming with Fixpoint Types by Rob Norris](https://www.youtube.com/watch?v=7xSfLPD6tiQ)
2. [Going bananas with recursion schemes for fixed point data types by Paweł Szulc](https://www.youtube.com/watch?v=IlvJnkWH6CA)
3. [F-Algebra by 夏梓耀](https://zhuanlan.zhihu.com/p/21354189)