---
layout: post
title:  F-bounded type polymorphism
date:   2019-03-19 23:06:30 -0700
categories: 
    - Programming
    - Scala 
---
# F-bounded type polymorphism

当我第一次看到别人是向下面代码这样给出Java 中 Functor的定义的时候，我是一脸懵逼的，因为完全不明白在type parameter 里的是什么，为什么要这么写
```java
interface Functor<T, F extends Functor<?, F>> {
    <R> F map(Function<? extends T, R> f);
}
```

虽然这样定义出来的Functor是残疾的,在返回的F那里type信息丢失了，但是这不是定义的锅，而是java的锅。谁让它没有higher kinded types。


偏题了，今天想记录的是F-bounded type polymorphism，就是Functor<T, F extends Functor<?, F>>这块，一个从java5 开始就有了的feature。

## 例子

假设在一个billing系统里有多个payment type，程序员某奇发现在公司的屎山代码里发现有很多类似如下的类定义：
```java
/* AlipayAccount.class */
public class AlipayAccount {
    public AlipayAccount init() {
        ...
    }

    public AlipayAccount load(Long id) {
        ...
    }

    public void update(AlipayAccount account) {
        ...
    }

    public void delete(Long id) {
        ...
    }
}
```

```java
/* PaypalAccount.class */
public class PaypalAccount {
public PaypalAccount init() {
        ...
    }

    public PaypalAccount load(Long id) {
        ...
    }

    public void update(PaypalAccount account) {
        ...
    }

    public void delete(Long id) {
        ...
    }
}
```

不懂事情的某奇由于图样图森破想要去refactor这些代码，因为这些代码看上去完全没差。

某奇可能会这么去refactor
```java
/* Account.class */
public interface Account<T> {
    T init();
    T load(Long id);
    void update(T account);
    void delete(Long id);
}

/* AlipayAccount.class */
public class AlipayAccount extends Account<AlipayAccount> {
    ...
}

/* PaypalAccount.class */
public class PaypalAccount extends Account<PaypalAccount> {
    ...
}
```

但是，这样不够好，为什么？ 在继承的时候Account的type parameter可能会与类名不一致。Account并没有对type parameter做类型上的限制

```java
public class DummyAccount extends Account<PaypalAccount> {
    ...
}
```

这就是F bounded type parameter的发挥用处的时候了。如果用了F bounded type parameter，在任何不是 E extends Account\[E\]的情况， 编译器都会出错。
下面这样的代码就会出错：

```java
abstract class Account<T extends Account<T>> {
    abstract T load(long id);
}

public class Paypal extends Account<Alipay> { //Alipay is not within bound

    @Override
    public Alipay load(long id) {
        return null;
    }
}

public class Alipay extends Account<Paypal> { //Paypal is not within bound
    @Override
    public Paypal load(long id) {
        return null;
    }
}
```

## 好处都有啥
安全！

其实不去碰屎山更安全(ε=ε=ε=┏(゜ロ゜;)┛


