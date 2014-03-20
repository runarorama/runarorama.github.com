---
layout: post
title: "Monoid morphisms, products, and coproducts"
date: 2014-03-19
comments: true
categories: 
author: Rúnar
commentIssueId: 8
---

Today I want to talk about relationships between monoids. These can be useful to think about when we're developing libraries involving monoids, and we want to express some algebraic laws among them. We can then check these with automated tests, or indeed _prove_ them with algebraic reasoning.

This post kind of fell together when writing notes on chapter 10, "Monoids", of [Functional Programming in Scala](http://manning.com/bjarnason). I am putting it here so I can reference it from the chapter notes at the end of the book.

## Monoid homomorphisms ##

Let's take the `String` concatenation and `Int` addition as example monoids that have a relationship. Note that if we take the length of two strings and add them up, this is the same as concatenating those two strings and taking the length of the combined string:

{% codeblock lang:scala %}
(a.length + b.length) == (a + b).length
{% endcodeblock %}

So every `String` maps to a corresponding `Int` (its length), and every concatenation of strings maps to the addition of corresponding lengths.

The `length` function maps from `String` to `Int` _while preserving the monoid structure_. Such a function, that maps from one monoid to another in such a preserving way, is called a _monoid homomorphism_. In general, for monoids `M` and `N`, a homomorphism `f: M => N`, and all values `x:M`, `y:M`, the following law holds:

{% codeblock lang:scala %}
f(x |+| y) == (f(x) |+| f(y))
{% endcodeblock %}

The `|+|` syntax is from [Scalaz](http://github.com/scalaz/scalaz) and is obtained by importing `scalaz.syntax.monoid._`. It just references the `append` method on the `Monoid[T]` instance, where `T` is the type of the arguments.

This law can have real practical benefits. Imagine for example a "result set" monoid that tracks the locations of a particular set of records in a database or file. This could be as simple as a `Set` of locations. Concatenating several thousand files and then proceeding to search through them is going to be much slower than searching through the files individually and then concatenating the result sets. Particularly since we can potentially search the files in parallel. A good automated test for our result set monoid would be that it admits a homomorphism from the data file monoid.

## Monoid isomorphisms ##

Sometimes there will be a homomorphism in both directions between two monoids. If these are inverses of one another, then this kind of relationship is called a _monoid isomorphism_ and we say that the two monoids are isomorphic. More precisely, we will have two monoids `A` and `B`, and homomorphisms `f: A => B` and `g: B => A`. If `f(g(b)) == b` and `g(f(a)) == a`, for all `a:A` and `b:B` then `f` and `g` form an isomorphism.

For example, the `String` and `List[Char]` monoids with concatenation are isomorphic. We can convert a `String` to a `List[Char]`, preserving the monoid structure, and go back again to the exact same `String` we started with. This is also true in the inverse direction, so the isomorphism holds.

Other examples include (`Int`, `+`) and (`Int`, `*`), which are isomorphic. The same goes for (`Boolean`, `&&`) and (`Boolean`, `||`).

Note that there are monoids with homomorphisms in both directions between them that nevertheless are _not_ isomorphic.

## Monoid products and coproducts ##

If `A` and `B` are monoids, then `(A,B)` is certainly a monoid, called their _product_:

{% codeblock lang:scala %}
def product[A:Monoid,B:Monoid]: Monoid[(A, B)] =
  new Monoid[(A, B)] {
    def append(x: (A, B), y: => (A, B)) =
      (x._1 |+| y._1, x._2 |+| y._2)
    val zero = (mzero[A], mzero[B])
  }
{% endcodeblock %}

But is there such a thing as a monoid _coproduct_? It's certainly not possible to build a monoid on `Either[A,B]` for monoids `A` and `B`. For example, what would be the `zero` of such a monoid? And what would be the value of `Left(a) |+| Right(b)`? We could certainly choose an arbitrary rule, but not one that actually satisfies the monoid laws of associativity and unit.

To resolve this, we need to know the precise meaning of _product_ and _coproduct_. These come straight from Wikipedia, with a little help from Cale Gibbard.

A _product_ `M` of two monoids `A` and `B` is a monoid such that there exist homomorphisms `fst: M => A`, `snd: M => B`, and for any monoid `Z` and morphisms `f: Z => A` and `g: Z => B` there has to be a unique homomorphism `h: Z => M` such that `fst(h(z)) == f(z)` and `snd(h(z)) == g(z)` for all `z:Z`. In other words, the following diagram must commute:

{% img /images/Product.png %}

A _coproduct_ `W` of two monoids `A` and `B` is the same except the arrows are reversed. It's a monoid such that there exist homomorphisms `left: A => W`, `right: B => W`, and for any monoid `Z` and morphisms `f: A => Z` and `g: B => Z` there has to be a unique homomorphism `h: W => Z` such that `h(left(a)) == f(a)` and `h(right(b)) == g(b)` for all `a:A` and all `b:B`. In other words, the following diagram must commute:

{% img /images/Coproduct.png %}

We can easily show that our `productMonoid` above really is a monoid product. The homomorphisms are the methods `_1` and `_2` on `Tuple2`. They simply map every element of `(A,B)` to a corresponding element in `A` and `B`. The monoid structure is preserved because:

{% codeblock lang:scala %}
(p |+| q)._1 == (p._1 |+| q._1)
(p |+| q)._2 == (p._2 |+| q._2)
{% endcodeblock %}

And for any other monoid `Z`, and morphisms `f: Z => A` and `g: Z => B`, we can construct a unique morphism from `Z` to `(A,B)`:

{% codeblock lang:scala %}
def factor[A:Monoid,B:Monoid,Z:Monoid](z: Z)(f: Z => A, g: Z => B): (A, B) =
  (f(z), g(z))
{% endcodeblock %}

And this really is a homomorphism because we just inherit the homomorphism law from `f` and `g`.

What does a coproduct then look like? Well, it's going to be a type `C[A,B]` together with an instance `coproduct[A:Monoid,B:Monoid]:Monoid[C[A,B]]`. It will be equipped with two monoid homomorphisms, `left: A => C[A,B]` and `right: B => C[A,B]` that satisfy the following (according to the monoid homomorphism law):

{% codeblock lang:scala %}
(left(a1) |+| left(a2)) == left(a1 |+| a2)
(right(b1) |+| right(b2)) == right(b1 |+| b2)
{% endcodeblock %}

And additionally, for any other monoid `Z` and homomorphisms `f: A => Z` and `g: B => Z` we must be able to construct a unique homomorphism from `C[A,B]` to `Z`:

{% codeblock lang:scala %}
def fold[A:Monoid,B:Monoid,Z:Monoid](c: C[A,B])(f: A => Z, g: B => Z): Z
{% endcodeblock %}


### The simplest thing that could possibly work ###

We can simply come up with a data structure required for the coproduct to satisfy a monoid. We will start with two constructors, one for the left side, and another for the right side:

{% codeblock lang:scala %}
sealed trait These[+A,+B]
case class This[A](a: A) extends These[A,Nothing]
case class That[B](b: B) extends These[Nothing,B]
{% endcodeblock %}

This certainly allows an embedding of both monoids `A` and `B`. But `These` is now basically `Either`, which we know doesn't quite form a monoid. What's needed is a `zero`, and a way of appending an `A` to a `B`. The simplest way to do that is to add a product constructor to `These`:

{% codeblock lang:scala %}
case class Both[A,B](a: A, b: B) extends These[A,B]
{% endcodeblock %}

Now `These[A,B]` is a monoid as long as `A` and `B` are monoids:

{% codeblock lang:scala %}
def coproduct[A:Monoid,B:Monoid]: Monoid[These[A, B]] =
  new Monoid[These[A, B]] {
    def append(x: These[A, B], y: These[A, B]) = (x, y) match {
      case (This(a1), This(a2)) => This(a1 |+| a2)
      case (That(b1), That(b2)) => That(b1 |+| b2)
      case (That(b), This(a)) => Both(a, b)
      case (This(a), That(b)) => Both(a, b)
      case (Both(a1, b), This(a)) => Both(a1 |+| a, b)
      case (Both(a, b1), That(b)) => Both(a, b1 |+| b)
      case (This(a1), Both(a, b)) => Both(a1 |+| a, b)
      case (That(b), Both(a, b1)) => Both(a, b1 |+| b)
      case (Both(a1, b1), Both(a2, b2)) => Both(a1 |+| a2, b1 |+| b2)
    }
    val zero = Both(A.zero, B.zero)
  }
{% endcodeblock %}

`These[A,B]` is the smallest monoid that contains both `A` and `B` as submonoids (the `This` and `That` constructors, respectively) and admits a homomorphism from both `A` and `B`. And notice that we simply added the least amount of structure possible to make `These[A,B]` a monoid (the `Both` constructor). But is it really a coproduct?

First we must prove that `This` and `That` really are homomorphisms. We need to prove the following two properties:

{% codeblock lang:scala %}
(This(a1) |+| This(a2)) == This(a1 |+| a2)
(That(b1) |+| That(b2)) == That(b1 |+| b2)
{% endcodeblock %}

That's easy. The first two cases of the `append` method on the `coproduct` monoid prove these properties.

But can we define `fold`? Yes we can:

{% codeblock lang:scala %}
def fold[A:Monoid,B:Monoid,Z:Monoid](
  these: These[A,B])(f: A => Z, g: B => Z): Z =
    these match {
      case This(a) => f(a)
      case That(b) => g(b)
      case Both(a, b) => f(a) |+| g(b)
    }
{% endcodeblock %}

But is `fold` really a homomorphism? Let's not assume that it is, but test it out.Here's the homomorphism law:

{% codeblock lang:scala %}
fold(f,g)(t1 |+| t2) == fold(f,g)(t1) |+| fold(f,g)(t2)
{% endcodeblock %}

What happens if both `t1` or `t2` are `This` or `That`?

{% codeblock lang:scala %}
fold(f,g)(This(a1) |+| This(a2)) ==
fold(f,g)(This(a1)) |+| fold(f, g)(This(a2))

f(a1) |+| f(a2) == f(a1) |+| f(a2)
{% endcodeblock %}

That holds. But what if we introduce a `Both` on one side?

{% codeblock lang:scala %}
fold(f,g)(This(a1) |+| Both(a2, b)) ==
fold(f,g)(This(a1)) |+| fold(f,g)(Both(a2,b))

f(a1 |+| a2) |+| g(b) == f(a1) |+| f(a2) |+| g(b)
{% endcodeblock %}

So far so good. That holds because of associativity. What about the other side?

{% codeblock lang:scala %}
fold(f,g)(That(b1) |+| Both(a, b2)) ==
fold(f,g)(That(b1)) |+| fold(f,g)(Both(a,b2))

f(a) |+| g(b1 |+| b2) == g(b1) |+| f(a) |+| g(b2)
{% endcodeblock %}

No! Something has gone wrong. This will only hold if the `Z` monoid is commutative. So in general, `These[A,B]` is not the coproduct of `A` and `B`. My error was in the `Both` constructor, which commutes all `B` values to the right and all `A` values to the left.

So what kind of thing would work? It would have to solve this case:

{% codeblock lang:scala %}
  case (Both(a1, b1), Both(a2, b2)) => ???
{% endcodeblock %}

We need to preserve that `a1`, `b1`, `a2`, and `b2` appear _in that order_. So clearly the coproduct will be some kind of list!


### Free monoids on coproducts ###

Let's try going the other way. What if we start with the coproduct of the underlying sets and get a free monoid from there?

The underlying set of a monoid `A` is just the type `A` without the monoid structure. The coproduct of types `A` and `B` is the type `Either[A,B]`. Having "forgotten" the monoid structure of both `A` and `B`, we can recover it by generating a free monoid on `Either[A,B]`, which is just `List[Either[A,B]]`. The `append` operation of this monoid is list concatenation, and the identity for it is the empty list.

Clearly `List[Either[A,B]]` is a monoid, but does it permit a homomorphism from both monoids `A` and `B`? If so, then the following properties should hold:

{% codeblock lang:scala %}
List(Left(a1)) ++ List(Left(a2))) == List(Left(a1 |+| a2))
List(Right(b1)) ++ List(Right(b2))) == List(Right(b1 |+| b2))
{% endcodeblock %}

They clearly do not hold! The lists on the left of `==` will have two elements and the lists on the right will have one element. Can we do something about this?

Well, the fact is that `List[Either[A,B]]` is not exactly the monoid coproduct of `A` and `B`. It's "too big" in a sense. But if we were to reduce the list to a normal form that approximates a "free product", we can get a coproduct that matches our definition above. What we need is a new monoid:

{% codeblock lang:scala %}
sealed class Eithers[A,B](val toList: List[Either[A,B]]) {
  def ++(p: Eithers[A,B]): Eithers[A,B] =
    Eithers(toList ++ p.toList)
}

object Eithers {
  def apply[A:Monoid,B:Monoid](xs: List[Either[A,B]]): Eithers[A,B] =
    xs.foldRight(List[Either[A,B]]()) {
      case (Left(a1), Left(a2) :: xs) => Left(a1 |+| a2) :: xs
      case (Right(b1), Right(b2) :: xs) => Right(b1 |+| b2) :: xs
      case (e, xs) => e :: xs
    }
  def empty[A,B]: Eithers[A,B] = new Eithers(Nil)
}
{% endcodeblock %}

`Eithers[A,B]` is a kind of `List[Either[A,B]]` that has been normalized so that consecutive `A`s and consecutive `B`s have been collapsed using their respective monoids. So it will contain alternating `A` and `B` values.

This is now a monoid coproduct because it permits monoid homomorphisms from `A` and `B`:

{% codeblock lang:scala %}
Eithers(List(Left(a1))) ++ Eithers(List(Left(a2)))) ==
  Eithers(List(Left(a1 |+| a2)))
Eithers(List(Right(b1))) ++ Eithers(List(Right(b2)))) ==
  Eithers(List(Right(b1 |+| b2)))
{% endcodeblock %}

And we can implement the `fold` homomorphism:

{% codeblock lang:scala %}
def fold[A:Monoid,B:Monoid,Z:Monoid](
  es: Eithers[A,B])(f: A => Z, g: B => Z): Z =
    es.toList.foldRight(mzero[Z]) {
      case (Left(a), z) => f(a) |+| z
      case (Right(b), z) => g(b) |+| z
    }
{% endcodeblock %}

And this time `fold` really is a homomorphism, and we can prove it by case analysis. Here's the law again:

{% codeblock lang:scala %}
(fold(e1)(f,g) |+| fold(e2)(f,g)) == fold(e1 ++ e2)(f,g)
{% endcodeblock %}

If either of `e1` or `e2` is empty then the result is the fold of the other, so those cases are trivial. If they are both nonempty, then they will have one of these forms:

{% codeblock lang:scala %}
e1 = [..., Left(a1)]
e2 = [Left(a2), ...]

e1 = [..., Right(b1)]
e2 = [Right(b2), ...]

e1 = [..., Left(a)]
e2 = [Right(b), ...]

e1 = [..., Right(b)]
e2 = [Left(a), ...]
{% endcodeblock %}

In the first two cases, on the right of the `==` sign in the law, we perform `a1 |+| a2` and `b1 |+| b2` respectively before concatenating. In the other two cases we simply concatenate the lists. The `++` method on `Eithers` takes care of doing this correctly for us. On the left of the `==` sign we fold the lists individually and they will be alternating applications of `f` and `g`. So then this law amounts to the fact that `f(a1 |+| a2) == f(a1) |+| f(a2)` in the first case, and the same for `g` in the second case. In the latter two cases this amounts to a homomorphism on `List`. So as long as `f` and `g` are homomorphisms, so is `fold(_)(f,g)`. Therefore, `Eithers[A,B]` really is a coproduct of `A` and `B`.

The lesson learned here is to check assumptions and test against laws. Things are not always as straightforward as they seem.

