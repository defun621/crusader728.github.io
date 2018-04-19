---
layout: post
title:  Scala中的implicit
date:   2018-04-19 01:05:30 -0700
categories: Scala
---
最近在公司里和自己业余的时候重新上手Scala，在这里记录下来自己在写的时候学到的东西。之前虽然有Chris指路，但是我只是浅尝则止罢了。希望今次可以走得远一点，学得深一点。

回到本篇的主题，Scala的implicit功能很强大，好的implicit可以让代码更优雅，通过方法注入的方式实现类似于C#扩展方法的功能。

举例来说，现在在我写的简单Scala代码中，我想简单地通过Jsoup库拿到一个URL所对应的页面，即：
···Scala
new URL("www.google.com").fetch
···

