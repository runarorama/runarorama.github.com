---
layout: post
title: "Free monoids and free monads"
date: 2013-08-20
comments: true
categories:
author: Rúnar
commentIssueId: 6
---

In a series of old posts, I once talked about the link between [lists and monoids](http://apocalisp.wordpress.com/2010/06/14/on-monoids/), as well as [monoids and monads](http://apocalisp.wordpress.com/2010/07/21/more-on-monoids-and-monads/). Now I want to talk a little bit more about monoids and monads from the perspective of _free structures_.

`List` is a _free monoid_. That is, for any given type `A`, `List[A]` is a monoid, with list concatenation as the operation and the empty list as the identity element.

{% codeblock lang:scala %}
def freeMonoid[A]: Monoid[List[A]] = new Monoid[List[A]] {
  def unit = Nil
  def op(a1: List[A], a2: List[A]) = a1 ++ a2
}
{% endcodeblock %}

Being a free monoid means that it's the _minimal_ such structure. `List[A]` has exactly enough structure so that it is a monoid for any given `A`, and it has no further structure. This also means that for any given monoid `B`, there must exist a transformation, a _monoid homomorphism_ from `List[A]` to `B`:

{% codeblock lang:scala %}
def foldMap[A,B:Monoid](as: List[A])(f: A => B): B
{% endcodeblock %}

Given a mapping from `A` to a monoid `B`, we can collapse a value in the monoid `List[A]` to a value in `B`.

## Free monads ##

Now, if you followed my old posts, you already know that monads are "higher-kinded monoids". A monoid in a category where the objects are type constructors (functors, actually) and the arrows between them are natural transformations. As a reminder, a natural transformation from `F` to `G` can be represented this way in Scala:

{% codeblock lang:scala %}
trait ~>[F[_],G[_]] {
  def apply[A](a: F[A]): G[A]
}
{% endcodeblock %}

And it turns out that there is a free monad for any given functor `F`:

{% codeblock lang:scala %}
sealed trait Free[F[_],A]
case class Return[F[_],A](a: A) extends Free[F,A]
case class Suspend[F[_],A](f: F[Free[F,A]]) extends Free[F,A]
{% endcodeblock %}

Analogous to how a `List[A]` is either `Nil` (the empty list) or a product of a `head` element and `tail` list, a value of type `Free[F,A]` is either an `A` or a product of `F[_]` and `Free[F,_]`. It is a recursive structure. And indeed, it has _exactly enough structure_ to be a monad, for any given `F`, and no more.

When I say "product" of two functors like `F[_]` and `Free[F,_]`, I mean a product like this:

{% codeblock lang:scala %}
trait :*:[F[_],G[_]] {
  type λ[A] = F[G[A]]
}
{% endcodeblock %}

So we might expect that there is a _monad homomorphism_ from a free monad on `F` to any monad that `F` can be transformed to. And indeed, it turns out that there is. The free monad catamorphism is in fact a monad homomorphism. Given a natural transformation from `F` to `G`, we can collapse a `Free[F,A]` to `G[A]`, just like with `foldMap` when given a function from `A` to `B` we could collapse a `List[A]` to `B`.

{% codeblock lang:scala %}
def runFree[F[_],G[_]:Monad,A](as: Free[F,A])(f: F ~> G): G[A]
{% endcodeblock %}

But what's the equivalent of `foldRight` for `Free`? Remember, foldRight takes a unit element `z` and a function that accumulates into `B` so that `B` doesn't actually have to be a monoid. Here, `f` is a lot like the monoid operation, except it takes the current `A` on the left:

{% codeblock lang:scala %}
def foldRight[A,B](as: List[A])(z: B)(f: (A, B) => B): B
{% endcodeblock %}

The equivalent for `Free` takes a natural transformation as its unit element, which for a monad happens to be monadic `unit`. Then it takes a natural transformation as its `f` argument, that looks a lot like monadic `join`, except it takes the current `F` on the left:

{% codeblock lang:scala %}
type Id[A] = A

def foldFree[F[_]:Functor,G[_],A](
  as: Free[F,A])(
  z: Id ~> G)(
  f: (F:*:G)#λ ~> G): G[A]
{% endcodeblock %}

In this case, `G` does not have to be a monad at all.

`Free` as well as natural transformations and product types are available in [Scalaz](http://github.com/scalaz).

