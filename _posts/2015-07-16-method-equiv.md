---
layout: post
title: When are two methods alike?

meta:
  nav: blog
  author: S11001001
  pygments: true
---

When are two methods alike?
===========================

We just saw two method types that, though different, are effectively
the same: those of `plengthT` and `plengthE`.  We have rules for
deciding when an existential parameter can be lifted into a method
type parameter—or a method type parameter lowered to an
existential—but there are other pairs of method types we’ll explore
that are the same, or very close.  So let’s talk about how we
determine this equivalence.

A method *r* is more general than or as general as *q* if *q* may be
implemented by only making a call to *r*, passing along the arguments.
By more general, we mean *r* can be invoked in all the situations that
*q* can be invoked in, and more besides.  Let us call the result of
this test *r* <:ₘ *q*, where <:ₘ is pronounced "party duck"; if the test
of *q* making a call to *m* fails, then *r* !<:ₘ *q*.

If *q* <:ₘ *r* and *r* <:ₘ *q*, then the two method types are
*equivalent*; that is, neither has more expressive power than the
other, since each can be implemented merely by invoking the other and
doing nothing else.  We write this as *q* ≡ₘ *r*.  Likewise, if *m* <:ₘ
*q* and *q* !<:ₘ *m*, that is, *q* can be written by calling *m*, but
not vice versa, then *m* is *strictly more general* than *q*.

What the concrete method—the one actually doing stuff, not invoking
the other one—does is irrelevant, for the purposes of this test,
because this is about types.  That matters because sometimes, in
Scala, as in Java, the body will compile in one of the methods, but
not the other.  Let’s see an example, that doesn’t compile.

```scala
import scala.collection.mutable.ArrayBuffer

def copyToZero(xs: ArrayBuffer[_]): Unit =
  xs += xs(0)
```

Likewise, the Java version has a similar problem.

```java
import java.util.List;

void copyToZero(final List<?> xs) {
    xs.add(xs.get(0));
}
```

Luckily, in both Java and Scala, we have an *equivalent* method type,
from lifting the existential (misleadingly called *wildcard* in Java
terminology) to a method type parameter.

We can apply this to put the method implementation somewhere it will
compile.

```scala
def copyToZeroT(xs: ArrayBuffer[_]): Unit =
  copyToZeroP(xs)

private def copyToZeroP[T](xs: ArrayBuffer[T]): Unit =
  xs += xs(0)
```

Similarly, in Java,

```java
import java.util.List;

void copyToZeroT(final List<?> xs) {
    copyToZeroP(xs);
}

void <T> copyToZeroP(final List<T> xs) {
    final T zv = xs.get(0);
    xs.add(zv);
}
```

The last gives a hint as to what’s going on: in `copyToZeroP`’s body,
the list element type has a name; we can use the name to create
variables, and the compiler can rely on the name as well.  The
compiler shouldn’t care about whether the name can be written, but
that one of the above compiles and the other doesn’t is telling.  In
both cases, we are just helping the compiler along by using the
equivalent method type; both `scalac` and `javac` manage to infer that
the type `T` should be the (otherwise unspeakable) existential.

When are two methods less alike?
--------------------------------

Now, let’s examine another pair of methods, and apply our test to
them.

Let’s say we want to write the equivalent of this method for `MList`.

```scala
def pdropFirst[T](xs: PList[T]): PList[T] =
  xs match {
    case PNil() => PNil()
    case PCons(_, t) => t
  }
```

According to the conversion rules given above, the equivalent for
`MList` should be

```scala
def mdropFirstT[T0](xs: MList {type T = T0})
  : MList {type T = T0} =
  xs.uncons match {
    case None => MNil()
    case Some(c) => c.tail
  }
```

Let us try to drop the refinements.  That seems to compile:

```scala
def mdropFirstE(xs: MList): MList =
  xs.uncons match {
    case None => MNil()
    case Some(c) => c.tail
  }
```

It certainly looks nicer.  However, while `mdropFirstE` can be
implemented by calling `mdropFirstT`, the opposite is not true;
`mdropFirstT` <m `mdropFirstE`, or, `mdropFirstT` is *strictly more
general*.

In this case, the reason is that `mdropFirstE` fails to relate the
argument’s `T` to the result’s `T`; you could implement `mdropFirstE`
like follows:

```scala
def mdropFirstE[T0](xs: MList): MList =
  MCons(42, MNil())
```

The stronger type of `mdropFirstT` forbids such shenanigans.  However,
I can tell you that largely because I’m already comfortable with
existentials; how could you figure that out if you’re just starting
out with these tools?  You don’t have to; the beauty of the
equivalence test is that you can apply it mechanically.  **Knowing
nothing about the mechanics of the parameterization and existentialism
of the types involved, you can work out with the equivalence test**
that `mdropFirstT` <m `mdropFirstE`, and so you can’t get away with
simply dropping the refinements.

Method likeness and subtyping, all alike
----------------------------------------

If you know what the symbol `<:` means in Scala, or perhaps you’ve
read SLS §3.5 "Relations between types" (TODO link), you might think,
"gosh, method equivalence and generality looks awfully familiar."

Indeed, the thing we’re talking about is very much like subtyping and
type equality!  In fact, every type-equal pair of methods m1 and m2
also pass our method equivalence test, and every pair of methods m3
and m4 where m3 <: m4 passes our m4-calls-m3 test.  So m1 ≡ m2 implies
m1 ≡ₘ m2, and m3 <: m4 implies m3 <:ₘ m4.

We even follow many of the same rules as the type relations.  We have
transitivity: if m1 can call m2 to implement itself, and m2 can call
m3 to implement itself, obviously we can snap the pointer and have m1
call m3 directly.  Likewise, every method type is equivalent to
itself: reflexivity.  Likewise, if a method m1 is strictly more
general than m2, obviously m2 cannot be strictly more general than m1:
antisymmetricity.  And we even copy the relationship between = and <:
themselves: just as t1 ≡ t2 implies t1 <: t2, so r ≡ₘ q implies r <:ₘ
q.

Scala doesn’t understand the notion of method equivalence we’ve
defined above, though.  So you can’t, say, implement an abstract
method in a subclass using an equivalent form, at least directly; you
have to `override` the Scala way, and call the alternative form
yourself, if that’s what you want.

I do confess to one oddity in my terminology: **the method that has
more specific type is *the more general method*.** I hope the example
of `mdropFirstT` <:m `mdropFirstE` justifies my choice.  `mdropFirstT`
is more specific, and rejects more implementations, such as the one
that returns a list with `42` in it above.  Thus, it has fewer
implementations, in the same way that more specific types have fewer
values inhabiting them.  But it can be used in more circumstances, so
it is "more general".  The generality in terms of when a method can be
used is directly proportional to the specificity of its type.

Java’s edge of insanity
-----------------------

Now we have enough power to demonstrate that Scala’s integration with
Java generics is faulty.  Or, more likely, that Java’s generics are
faulty.

Consider this method type, in Scala:

```scala
def goshWhatIsThis[T](t: T): T
```

This is a pretty specific method type; there are not too many
implementations.  Of course you can always perform a side effect; we
don’t track that in Scala’s type system.  But what can it return?
Just `t`.

Specifically, you can’t return `null`:

TODO error

Well now, let’s convert this type to Java:

```java
public <T> T holdOnNow(T t) {
    return null;
}
```

We got away with that!  And, indeed, we can call `holdOnNow` to
implement `goshWhatIsThis`, and vice versa; they’re *equivalent*.  But
the type says we can’t return `null`!

The problem is that Java adds an implicit upper bound, because it
assumes generic type parameters can only have class types chosen for
them; in Scala terms, `[T <: Object]`.  If we write that into the Java
code explicitly, Scala gives us the correct error.

```java
public <T extends Object> T holdOnNow(T t) {
    return null;
}
```

TODO error

This is forgivable on Scala’s part, because it’d be annoying to add
`<: AnyRef` to your generic methods just because you called some Java
code and it’s probably going to work out fine.  I blame `null`, and
while I’m at it, I blame `Object` having any methods at all, too.
We’d be better off without these bad features.