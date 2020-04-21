---
layout: post
title:  Church number 的Java 实现
date:   20120-04-21 11:15:30 -0700
categories: 
  - Programming
---
个人的邱奇数实现。自我感觉比[Rosetta Code上的Java实现](http://rosettacode.org/wiki/Church_Numerals#Java)好懂.
```java
public abstract class ChurchNumber {

    abstract <T> T apply(Function<T, T> f, T value);

    static final class CZero extends ChurchNumber {
        @Override
        <T> T apply(Function<T, T> f, T value) {
            return value;
        }
    }

    public static final ChurchNumber zero() {
        return new CZero();
    }

    public final ChurchNumber succ() {
        return new ChurchNumber() {
            @Override
            <T> T apply(Function<T, T> f, T value) {
                return f.apply(ChurchNumber.this.apply(f, value));
            }
        };
    }

    public final ChurchNumber plus(ChurchNumber a) {
        return new ChurchNumber() {
            @Override
            <T> T apply(Function<T, T> f, T value) {
                return a.apply(f, ChurchNumber.this.apply(f, value));
            }
        };
    }

    public final ChurchNumber multi(ChurchNumber a) {
        return new ChurchNumber() {
            @Override
            <T> T apply(Function<T, T> f, T value) {
                return a.apply(t -> ChurchNumber.this.apply(f, t), value);
            }
        };
    }

    public final ChurchNumber exp(ChurchNumber a) {
        return new ChurchNumber() {
            @Override
            <T> T apply(Function<T, T> f, T value) {
                return a.<Function<T, T>>apply(ttFunction -> t -> ChurchNumber.this.apply(ttFunction, t), f).apply(value);
            }
        };
    }

    public final ChurchNumber pred() {
        return steps().snd;
    }

    public final ChurchNumber minus(ChurchNumber a) {
        return a.apply(ChurchNumber::pred, this);
    }

    private static class Pair {
        ChurchNumber fst;
        ChurchNumber snd;

        Pair(ChurchNumber fst, ChurchNumber snd) {
            this.fst = fst;
            this.snd = snd;
        }
    }

    private static Pair step(Pair p) {
        return new Pair(p.fst.succ(), p.fst);
    }

    private Pair steps() {
        return ChurchNumber.this.apply(ChurchNumber::step, new Pair(zero(), zero()));
    }
}
```
