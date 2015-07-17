---
layout: post
title: Type projection isn't that specific

meta:
  nav: blog
  author: S11001001
  pygments: true
---

Type projection isn’t that specific
===================================

In the absence of the `Aux` trick presented at the end of
[the previous article]({% post_url 2015-07-19-forget-refinement-aux %}),
the continuous use of structural refinement to accomplish basic tasks
admittedly imposes a high cognitive load.  That is to say, it’s a lot
of work to say something that ought to be very simple.

Some people go looking for a solution, and find something that almost
seems to make sense: type projection (TODO pins or sls), or `MList#T`
in terms of our ongoing example (TODO 1st part link).  But **type
projection is, in almost all cases, too vague to really solve problems
you have using type members**.

A good reason to use type members
---------------------------------

Let’s see a simple example.  Here’s a sort of “value emitter”, that
operates in the space of some state, emitting a new value with each
step.

```scala
sealed abstract class StSource[A] {
  type S
  def init: S            // create the initial state
  def emit(s: S): (A, S) // emit a value, and update state
}

object StSource {
  type Aux[A, S0] = StSource[A] {type S = S0}

  def apply[A, S0](i: S0)(f: S0 => (A, S0)): Aux[A, S0] =
    new StSource[A] {
      type S = S0
      def init = i
      def emit(s: S0) = f(s)
    }
}
```

Unlike `MList`, there are actually good reasons to use type members
for the “state” in this sort of type definition; i.e. there are
reasonable designs in which you want to use member `S` existentially.
Thus, depending on how we intend to use it, it seems to meet our first
rule of thumb about when to use type members, as described in
[the first article of this series]({% post_url 2015-07-13-type-members-parameters %}).

A failed attempt at simplified emitting
----

So, under this theory, you’ve got some values of type `StSource[A]`
lying around.  And you want a simple function to take a source and its
state, and return a value and the new state.

```scala
def runStSource[A](ss: StSource[A], s: ??): (A, ??) = ss.emit(s)
```

But what do you put where the `??` is?  The surprising guess is often
`StSource[A]#S`.  After all, it means “the `StSource`’s `S`”, and
we’re trying to talk about an `StSource`’s `S`, right?

```scala
def runStSource[A](ss: StSource[A], s: StSource[A]#S)
  : (A, StSource[A]#S) = ss.emit(s)

TmTp4.scala:22: type mismatch;
 found   : s.type (with underlying type tmtp4.StSource[A]#S)
 required: ss.S
  : (A, StSource[A]#S) = ss.emit(s)
                                 ^
```

Setting aside that it won’t compile with the above signature -- the
usual outcome of experiments with type projection, that the types
aren’t strong enough to be workable without cheating by casting -- the
reality *sounds* so close to the above that it is understandable that
type projection is often confused with something useful.

<div class="side-note">
  There *are* uses for type projection.  But they are so rare, so
  exotic (they look like this TODO), and even the legitimate ones
  better off rewritten to avoid them, that the safer assumption is
  that you’ve gone down the wrong path if you’re trying to use them at
  all.  My suggestion can usually be phrased something like “move it
  to a companion object”.
</div>

In reality, `StSource[A]#S` means *some* `StSource`’s `S`.  Not the
one you gave, just any particular one.  That’s right, it’s
*existential*.  So, the failure of the above signature is like the
failure of `mdropFirstE` from section “When are two methods less
alike?” of [the second post of this series]({% post_url
2015-07-16-method-equiv %}): a failure to relate types strongly
enough.  The problem with `mdropFirstE` was failure to relate the
result type to argument type, whereas the failure above is to fail to
relate the two arguments’ types to each other.

Type parameters see existentially
----

As with `mdropFirstE`, a correct solution here is again lifting the
member to a method type parameter.

```scala
def runStSource[A, S](ss: StSource.Aux[A, S], s: S): (A, S) = ss.emit(s)
```

The surprising feature of this sort of signature is that it can be
invoked on `ss` arguments of type `StSource[A]`.

```
scala> val ss: StSource[Int] = StSource(0){i: Int => (i, i)}
ss: tmtp4.StSource[Int] = tmtp4.StSource$$anon$1@300b5011

scala> runStSource(ss, ss.init)
res0: (Int, ss.S) = (0,0)
```

In other words, **methods can assign names to unspecified, existential
type members**.  So even though we have a value whose type doesn’t
refine `S`, Scala still infers this type as the `S` argument to pass
to `runStSource`.

By analogy with type parameters, though, this isn’t too surprising.
We’ve already seen (TODO link) that `copyToZeroE` inferred its
argument’s existential parameter to pass along to the named parameter
to `copyToZeroP`, in the second part of this series.  We even saw it
apply directly to type members when `mdropFirstE` was able to invoke
`mdropFirstT`.  However, for whatever reason, we’re used to
existential parameters being able to do this; even Java manages the
task.  But it just seems *odder* that merely calling a method can fill
in the blanks in a whole refinement `{...}` raincloud.

It’s completely sound, though.  An `StSource` that exists as a value
*must* have an `S`, even if we existentialized it away.  So, as with
`_`s, let’s just give it a name to pass as the inferred type
parameter.  It makes a whole lot more sense than supposing
`StSource[A]#S` will just do what I mean.

In a future post, we’ll use this “infer the whole refinement” feature
to demonstrate that some of the most magical Scala type system
features aren’t really so magical.  But before we get to that, we need
to see just why existentials are anything but “wildcards”, and why it
doesn’t *always* make sense to be able to lift existentials like `S`
to type parameters.  That’s coming in the next post.