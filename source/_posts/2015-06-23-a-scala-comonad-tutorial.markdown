---
layout: post
title: "A Scala Comonad Tutorial"
date: 2015-06-23 13:15:47 -0400
comments: true
categories: 
---

In chapter 11 of [our book](http://manning.com/bjarnason), we talk about monads in Scala. This finally names a pattern that the reader has seen throughout the book and gives it a formal structure. We also give some intuition for what it _means_ for something to be a monad. Once you have this concept, you start recognizing it everywhere in the daily business of programming.

Today I want to talk about _comonads_, which are the dual of monads. The utility of comonads in everyday life is not quite as obviously applicable as that of monads, but they are a good tool to keep in your back pocket.

## A monad, upside-down

Let's remind ourselves of what a monad is. A monad is a functor, which just means it has a `map` method:

``` scala
trait Functor[F[_]] {
  def map[A,B](x: F[A])(f: A => B): F[B]
}
```

This has to satisfy the law that `map(x)(a => a) == x`, i.e. that mapping the identity function over our functor is a no-op.

### Monads

A monad is a functor `M` equipped with two additional polymorphic functions; One from `A` to `M[A]` and one from `M[M[A]]` to `M[A]`. 

``` scala
trait Monad[M[_]] extends Functor[M[_]] {
  def unit[A](a: A): M[A]
  def join[A](mma: M[M[A]]): M[A]
}
```

Recall that `join` has to satisfy associativity, and `unit` has to be an identity for `join`.

In Scala a monad is often stated in terms of `flatMap`, which is `map` followed by `join`. But I find this formulation easier to explain.

Every monad has the above operations, the so-called _proper morphisms_ of a monad, and may also bring to the table some _nonproper morphisms_ which give the specific monad some additional capabilities.

#### Reader monad

For example, the `Reader` monad brings the ability to ask for a value:

``` scala
case class Reader[R,A](run: R => A)

def ask[R]: Reader[R,R] = Reader(r => r)
```

The meaning of `join` in the reader monad is to pass the context of type `R` from the outer scope to the inner scope:

``` scala
def join[R,A](r: Reader[R,Reader[R,A]]) =
  Reader(c => r.run(c).run(c))
```

#### Writer monad

The `Writer` monad has the ability to write a value on the side:

``` scala
case class Writer[W,A](value: A, log: W)

def tell[A](w: W): Writer[W,Unit] = Writer((), w)
```

The meaning of `join` in the writer monad is to concatenate the "log" of written values using the monoid for `W`:

``` scala
def join[W:Monoid,A](w: Writer[W,Writer[W,A]]) =
  Writer(w.value.value, Monoid[W].append(w.log, w.value.log))
```

And the meaning of `unit` is to write the "empty" log:

``` scala
def unit[W:Monoid,A](a: A) = Writer(a, Monoid[W].zero)
```

#### State monad

The `State` monad can both get and set the state:

``` scala
def get[S]: State[S,S] = State(s => (s, s))
def put[S](s: S): State[S,Unit] = State(_ => ((), s))
```

The meaning of `join` in the state monad is to give the outer action an opportunity to get and put the state, then do the same for the inner action:

``` scala
case class State[S,A](run: S => (A, S))

def join[S,A](v1: State[S,State[S,A]]): State[S,A] =
  State(s1 => {
    val (v2, s2) = v1.run(s1)
    v2.run(s2)
  }
```

#### Option monad

The `Option` monad can terminate without an answer:

``` scala
def none[A]: Option[A] = None
```

That's enough examples of monads. Let's now turn to comonads.

### Comonads

A comonad is the same thing as a monad, only backwards:

``` scala
trait Comonad[W[_]] extends Functor[W[_]] {
  def counit[A](w: W[A]): A
  def duplicate[A](wa: W[A]): W[W[A]]
}
```

Note that counit is pronounced "co-unit", not "cow-knit". It's also sometimes called `extract` because it allows you to get a value of type `A` _out of_ a `W[A]`.

This also has to obey some laws. We'll get to those later on, but let's first look at some examples.

## The identity comonad

A simple and obvious comonad is the dumb wrapper (the identity comonad):

``` scala
case class Id[A](a: A) {
  def map[B](f: A => B): Id[B] = Id(f(a))
  def counit: A = a
  def duplicate: Id[Id[A]] = Id(this)
}
```

This one is also the identity _monad_. `Id` doesn't have any functionality other than the proper morphisms of the (co)monad and is therefore not terribly interesting.

## The reader comonad

There's a comonad with the same capabilities as the reader monad, namely that it can ask for a value:

``` scala
case class Coreader[R,A](extract: A, ask: R) {
  def map[B](f: A => B): Coreader[R,B] = Coreader(f(extract), ask)
  def duplicate: Coreader[R, Coreader[R, A]] =
    Coreader(this, ask)
}
```

It should be obvious how we can give a `Comonad` instance for this:

``` scala
def coreaderComonad[R]: Comonad[Coreader[R,?]] =
  new Comonad[Coreader[R,?]] {
    def map[A,BB](c: Coreader[R,A])(f: A => B) = c map f
    def counit[A](c: Coreader[R,A]) = c.extract
    def duplicate(c: Coreader[R,A]) = c.duplicate
  }
```

Arguably, this is much more straightforward in Scala than the reader monad. In the reader _monad_, the `ask` function is the identity function. That's saying "once the `R` value is available, return it to me", making it available to subsequent `map` and `flatMap` operations. But in `Coreader`, we don't have to pretend to have an `R` value. It's just right there and we can look at it.

So `Coreader` just wraps up some value of type `A` together with some additional context of type `R`. Why is it important that this is a _comonad_? What is the meaning of `duplicate` here?

The meaning of `duplicate` is that it puts the whole `Coreader` in the value slot. So any subsequent `extract` or `map` operation will be able to observe both the value of type `A` and the context of type `R`. We can think of this as passing the context along to those subsequent operations, which is analogous to what the reader monad does.

In fact, just like `map` followed by `join` is usually expressed as `flatMap`, by the same token `duplicate` followed by `map` is usually expressed as a single operation, `extend`:

``` scala
case class Coreader[R,A](extract: A, ask: R) {
  ...
  def extend[B](f: Coreader[R,A] => B): Coreader[R,B] =
    duplicate map f
}
```

Notice that the type signature of `extend` looks like `flatMap` with the direction of `f` reversed. And just like we can chain operations in a monad using `flatMap`, we can chain operations in a comonad using `extend`. In `Coreader`, `extend` is making sure that `f` can use the context of type `R` to produce its `B`. Chaining operations this way using `flatMap` or `extend` is sometimes called _Kleisli composition_.

If `R` is a monoid, then `Coreader[R,?]` can be the writer monad and the reader comonad simultaneously. Then actions in the comonad can ask for the current value of the `R`, and actions in the monad can add to it.

## The writer comonad

Just like the writer monad, the writer comonad can append to a log or running tally using a monoid. But instead of keeping the log always available to be appended to, it uses the same trick as the reader monad by building up an operation that gets executed once a log becomes available:

``` scala
class Cowriter[W:Monoid,A](tell: W => A) {
  def map[B](f: A => B): Cowriter[W,B] = Cowriter(tell andThen f)
  def extract = tell(Monoid[W].zero)
  def duplicate: Cowriter[W, Cowriter[W, A]] =
    Cowriter(w1 => Cowriter(w2 => tell(Monoid[W].append(w1, w2))))
  def extend[B](f: Cowriter[R,A] => B): Cowriter[R,B] =
    duplicate map f
}
```

Note that `duplicate` returns a whole `Cowriter` from its constructed `run` function, so the meaning is that subsequent operations (composed via `map` or `extend`) have access to exactly one `tell` function, which appends to the existing log or tally.

## The comonad laws

The comonad laws are analogous to the monad laws:

  1. Left idenity: `wa.duplicate.extract == wa`
  2. Right identity: `wa.extend(extract) == wa`
  3. Associativity: `wa.duplicate.duplicate == wa.extend(duplicate)`

It can be hard to get an intuition for what these laws _mean_, but in short they mean that (co)Kleisli composition in a comonad should be associative and that `extract` (a.k.a. `counit`) should be an identity for it.

Intuitively, both the monad and comonad laws mean that we should be able to write our programs top-down or bottom-up, or any combination thereof, and have that mean the same thing regardless.

## Next time...

In part 2 we'll look at some more examples of comonads and follow some of the deeper connections. Like what's the relationship between the reader monad and the reader comonad, or the writer monad and the writer comonad? They're not identical, but they seem to do all the same things. Are they equivalent? Isomorphic? Something else?


