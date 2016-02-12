---
layout: post
title: "Scala Comonad Tutorial, Part 2"
date: 2015-10-4 22:36:31 +0100
comments: true
author: Rúnar
categories: 
commentIssueId: 20
---

In the [previous post](http://blog.higher-order.com/blog/2015/06/23/a-scala-comonad-tutorial/), we looked at the Reader/Writer monads and comonads, and discussed in general what comonads are and how they relate to monads. This time around, we're going to look at some more comonads, delve briefly into adjunctions, and try to get some further insight into what it all means.

## Nonempty structures ##

Since a comonad has to have a `counit`, it must be "pointed" or nonempty in some sense. That is, given a value of type `W[A]` for some comonad `W`, we must be able to get a value of type `A` out.

The identity comonad is a simple example of this. We can always get a value of type `A` out of `Id[A]`. A slightly more interesting example is that of non-empty lists:

``` scala
case class NEL[A](head: A, tail: Option[NEL[A]])
```

So a nonempty list is a value of type `A` together with either another list or `None` to mark that the list has terminated. Unlike the traditional `List` data structure, we can always safely get the `head`.

But what is the comonadic `duplicate` operation here? That should allow us to go from `NEL[A]` to `NEL[NEL[A]]` in such a way that the comonad laws hold. For nonempty lists, an implementation that satisfies those laws turns out to be:

``` scala
case class NEL[A](head: A, tail: Option[NEL[A]]) {
  ...
  def tails: NEL[NEL[A]] =
    NEL(this, tail.map(_.tails))
  ...
}
```

The `tails` operation returns a list of all the suffixes of the given list. This list of lists is always nonempty, because the first suffix is the list itself. For example, if we have the nonempty list `[1,2,3]` (to use a more succinct notation), the `tails` of that will be `[[1,2,3], [2,3], [3]]`

To get an idea of what this _means_ in the context of a comonadic program, think of this in terms of coKleisli composition, or `extend` in the comonad:

``` scala
  def extend[A,B](f: NEL[A] => B): NEL[B] =
    tails map f
```

When we `map` over `tails`, the function `f` is going to receive each suffix of the list in turn. We apply `f` to each of those suffixes and collect the results in a (nonempty) list. So `[1,2,3].extend(f)` will be `[f([1,2,3]), f([2,3]), f([3])]`.

The name `extend` refers to the fact that it takes a "local" computation (here a computation that operates on a list) and extends that to a "global" computation (here over all suffixes of the list).

Or consider this class of nonempty trees (often called Rose Trees):

``` scala
case class Tree[A](tip: A, sub: List[Tree[A]])
```

A tree of this sort has a value of type `A` at the tip, and a (possibly empty) list of subtrees underneath. One obvious use case is something like a directory structure, where each `tip` is a directory and the corresponding `sub` is its subdirectories.

This is also a comonad. The `counit` is obvious, we just get the `tip`. And here's a `duplicate` for this structure:

``` scala
  def duplicate: Tree[Tree[A]] =
    Tree(this, sub.map(_.duplicate))
```

Now, this obviously gives us a tree of trees, but what is the structure of that tree? It will be _a tree of all the subtrees_. The `tip` will be `this` tree, and the `tip` of each proper subtree under it will be the entire subtree at the corresponding point in the original tree.

That is, when we say `t.duplicate.map(f)` (or equivalently `t extend f`), our `f` will receive each subtree of `t` in turn and perform some calculation over that entire subtree. The result of the whole expression `t extend f` will be a tree mirroring the structure of `t`, except each node will contain `f` applied to the corresponding subtree of `t`.

To carry on with our directory example, we can imagine wanting a detailed space usage summary of a directory structure, with the size of the whole tree at the `tip` and the size of each subdirectory underneath as tips of the subtrees, and so on. Then `d extend size` creates the tree of sizes of recursive subdirectories of `d`.

## The cofree comonad ##

You may have noticed that the implementations of `duplicate` for rose trees and `tails` for nonempty lists were basically identical. The only difference is that one is mapping over a `List` and the other is mapping over an `Option`. We can actually abstract that out and get a comonad for any functor `F`:

``` scala
case class Cofree[F[_],A](counit: A, sub: F[Cofree[F,A]]) {
  def duplicate(implicit F: Functor[F]): Cofree[F,Cofree[F,A]] =
    Cofree(this, F.map(sub)(_.duplicate))
}
```

A really common kind of structure is something like the type `Cofree[Map[K,?],A]` of trees where the `counit` is some kind of summary and each key of type `K` in the `Map` of subtrees corresponds to some drilldown for more detail. This kind of thing appears in portfolio management applications, for example.

Compare this structure with the _free monad_:

``` scala
sealed trait Free[F[_],A]
case class Return[F[_],A](a: A) extends Free[F,A]
case class Suspend[F[_],A](s: F[Free[F,A]]) extends Free[F,A]
```

While the free monad is _either_ an `A` or a recursive step suspended in an `F`, the cofree comonad is _both_ an `A` _and_ a recursive step suspended in an `F`. They really are duals of each other in the sense that the monad is a coproduct and the comonad is a product.

## Comparing comonads to monads (again) ##

Given this difference, we can make some statements about what it means:

* `Free[F,A]` is a type of "leafy tree" that branches according to `F`, with values of type `A` at the leaves, while `Cofree[F,A]` is a type of "node-valued tree" that branches according to `F` with values of type `A` at the nodes.
* If `Exp` defines the structure of some expression language, then `Free[Exp,A]` is the type of abstract syntax trees for that language, with free variables of type `A`, and monadic `bind` literally binds expressions to those variables. Dually, `Cofree[Exp,A]` is the type of _closed_ exresspions whose subexpressions are annotated with values of type `A`, and comonadic `extend` _reannotates_ the tree. For example, if you have a type inferencer `infer`, then `e extend infer` will annotate each subexpression of `e` with its inferred type.

This comparison of `Free` and `Cofree` actually says something about monads and comonads in general:

* All monads can model some kind of leafy tree structure, and all comonads can be modeled by some kind of node-valued tree structure.
* In a monad `M`, if `f: A => M[B]`, then `xs map f` allows us to take the values at the leaves (`a:A`) of a monadic structure `xs` and _substitute_ an entire structure (`f(a)`) for each value. A subsequent `join` then renormalizes the structure, eliminating the "seams" around our newly added substructures. In a _comonad_ `W`, `xs.duplicate` denormalizes, or exposes the substructure of `xs:W[A]` to yield `W[W[A]]`. Then we can map a function `f: W[A] => B` over that to get a `B` for each part of the substructure and _redecorate_ the original structure with those values. (See Uustalu and Vene's excellent paper [The Dual of Substitution is Redecoration](http://cs.ioc.ee/~tarmo/papers/sfp01-book.pdf) for more on this connection.)
* A monad defines a class of programs whose subexpressions are incrementally generated from the outputs of previous expressions. A comonad defines a class of programs that incrementally generate output from the substructure of previous expressions.
* A monad adds structure by consuming values. A comonad adds values by consuming structure.

## The relationship between Reader and Coreader ##

If we look at a Kleisli arrow in the `Reader[R,?]` comonad, it looks like `A => Reader[R,B]`, or expanded out: `A => R => B`. If we uncurry that, we get `(A, R) => B`, and we can go back to the original by currying again. But notice that a value of type `(A, R) => B` is a coKleisli arrow in the `Coreader` comonad! Remember that `Coreader[R,A]` is really a pair `(A, R)`.

So the answer to the question of how `Reader` and `Coreader` are related is that there is a one-to-one correspondence between a Kleisli arrow in the `Reader` monad and a coKleisli arrow in the `Coreader` comonad. More precisely, the Kleisli category for `Reader[R,?]` is isomorphic to the coKleisli category for `Coreader[R,?]`. This isomorphism is witnessed by currying and uncurrying.

In general, if we have an isomorphism between arrows like this, we have what's called an _adjunction_:

``` scala
trait Adjunction[F[_],G[_]] {
  def left[A,B](f: F[A] => B): A => G[B]
  def right[A,B](f: A => G[B]): F[A] => B
}
```

In an `Adjunction[F,G]`, we say that `F` is _left adjoint_ to `G`, often expressed with the notation `F ⊣ G`.

We can clearly make an `Adjunction` for `Coreader[R,?]` and `Reader[R,?]` by using `curry` and `uncurry`:

``` scala
def homSetAdj[R] = new Adjunction[(?, R), R => ?] {
  def left[A,B](f: ((A, R)) => B): A => R => B =
    Function.untupled(f).curried
  def right[A,B](f: A => R => B): ((A, R)) => B =
    Function.uncurried(f).tupled
}
```

The additional `tupled` and `untupled` come from the unfortunate fact that I've chosen Scala notation here and Scala differentiates between functions of two arguments and functions of one argument that happens to be a pair.

So a more succinct description of this relationship is that `Coreader` is left adjoint to `Reader`.

Generally the left adjoint functor _adds_ structure, or is some kind of "producer", while the right adjoint functor _removes_ (or "forgets") structure, or is some kind of "consumer".

## Composing adjoint functors ##

An interesting thing about adjunctions is that if you have an adjoint pair of functors `F ⊣ G`, then `F[G[?]]` always forms a comonad, and `G[F[?]]` always forms a monad, in a completely canonical and amazing way:

``` scala
def monad[F[_],G[_]](A: Adjunction[F,G])(implicit G: Functor[G]) =
  new Monad[λ[α => G[F[α]]]] {
    def unit[A](a: => A): G[F[A]] =
      A.left(identity[F[A]])(a)
    def bind[A,B](a: G[F[A]])(f: A => G[F[B]]): G[F[B]] =
      G.map(a)(A.right(f))
  }

def comonad[F[_],G[_]](A: Adjunction[F,G])(implicit F: Functor[F]) = 
  new Comonad[λ[α => F[G[α]]]] {
    def counit[A](a: F[G[A]]): A =
      A.right(identity[G[A]])(a)
    def extend[A,B](a: F[G[A]])(f: F[G[A]] => B): F[G[B]] =
      F.map(a)(A.left(f))
  }
```

Note that this says something about monads and comonads. Since the left adjoint `F` is a producer and the right adjoint `G` is a consumer, a monad always consumes and then produces, while a comonad always produces and then consumes.

Now, if we compose `Reader` and `Coreader`, which monad do we get?

```
scala> def M[S] = monad(homSetAdj[S])
M: [S]=> scalaz.Monad[[α]S => (α, S)]
```

That's the `State[S,?]` monad!

Now if we compose it the other way, we should get a comonad:

```
scala> def W[S] = comonad(homSetAdj[S])
W: [S]=> scalaz.Monad[[α](S => α, S)]
```

What is that? It's the `Store[S,?]` comonad:

``` scala
case class Store[S,A](peek: S => A, cursor: S) {
  def extract: A = peek(cursor)
  def extend[B](f: Store[S,A] => B): Store[S, B] =
    Store(s => f(Store(peek, s)), cursor)
  def duplicate: Store[S, Store[S,A]] =
    extend(identity)
  def map[B](f: A => B): Store[S,B] =
    extend(s => f(s.extract))

  def seek(s: S) = duplicate.peek(s)
}
```

This models a "store" of values of type `A` indexed by the type `S`. We have the ability to directly access the `A` value under a given `S` using `peek`, and there is a distinguished `cursor` or current position. The comonadic `extract` just reads the value under the `cursor`, and `duplicate` gives us a whole store full of stores such that if we `peek` at any one of them, we get a `Store` whose `cursor` is set to the given `s`. We're defining a `seek(s)` operation that moves the `cursor` to a given position `s` by taking advantage of `duplicate`.

A use case for this kind of structure might be something like image processing or cellular automata, where `S` might be coordinates into some kind of space (like a two-dimensional image). Then `extend` takes a local computation at the `cursor` and extends it to every point in the space. For example, if we have an operation `average` that peeks at the `cursor`'s immediate neighbors and averages them, then we can apply a low-pass filter to the whole image with `image.extend(average)`.

The type `A => Store[S,B]` is also one possible representation of a [Lens](http://docs.typelevel.org/api/scalaz/stable/7.1.0-M3/doc/#scalaz.package%24%24Lens%24). I might talk about lenses and [zippers](http://docs.typelevel.org/api/scalaz/stable/7.1.0-M3/doc/#scalaz.Zipper) in a future post.
