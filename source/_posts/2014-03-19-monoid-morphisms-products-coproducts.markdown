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

``` scala
(a.length + b.length) == (a + b).length
```

So every `String` maps to a corresponding `Int` (its length), and every concatenation of strings maps to the addition of corresponding lengths.

The `length` function maps from `String` to `Int` _while preserving the monoid structure_. Such a function, that maps from one monoid to another in such a preserving way, is called a _monoid homomorphism_. In general, for monoids `M` and `N`, a homomorphism `f: M => N`, and all values `x:M`, `y:M`, the following equations hold:

``` scala
f(x |+| y) == (f(x) |+| f(y))

f(mzero[M]) == mzero[N]
```

The `|+|` syntax is from [Scalaz](http://github.com/scalaz/scalaz) and is obtained by importing `scalaz.syntax.monoid._`. It just references the `append` method on the `Monoid[T]` instance, where `T` is the type of the arguments. The `mzero[T]` function is also from that same import and references `zero` in `Monoid[T]`.

This _homomorphism law_ can have real practical benefits. Imagine for example a "result set" monoid that tracks the locations of a particular set of records in a database or file. This could be as simple as a `Set` of locations. Concatenating several thousand files and then proceeding to search through them is going to be much slower than searching through the files individually and then concatenating the result sets. Particularly since we can potentially search the files in parallel. A good automated test for our result set monoid would be that it admits a homomorphism from the data file monoid.

## Monoid isomorphisms ##

Sometimes there will be a homomorphism in both directions between two monoids. If these are inverses of one another, then this kind of relationship is called a _monoid isomorphism_ and we say that the two monoids are isomorphic. More precisely, we will have two monoids `A` and `B`, and homomorphisms `f: A => B` and `g: B => A`. If `f(g(b)) == b` and `g(f(a)) == a`, for all `a:A` and `b:B` then `f` and `g` form an isomorphism.

For example, the `String` and `List[Char]` monoids with concatenation are isomorphic. We can convert a `String` to a `List[Char]`, preserving the monoid structure, and go back again to the exact same `String` we started with. This is also true in the inverse direction, so the isomorphism holds.

Other examples include (`Boolean`, `&&`) and (`Boolean`, `||`) which are isomorphic via `not`. 

Note that there are monoids with homomorphisms in both directions between them that nevertheless are _not_ isomorphic. For example, (`Int`, `*`) and (`Int`, `+`). These are homomorphic to one another, but not isomorphic (thanks, Robbie Gates).

## Monoid products and coproducts ##

If `A` and `B` are monoids, then `(A,B)` is certainly a monoid, called their _product_:

``` scala
def product[A:Monoid,B:Monoid]: Monoid[(A, B)] =
  new Monoid[(A, B)] {
    def append(x: (A, B), y: => (A, B)) =
      (x._1 |+| y._1, x._2 |+| y._2)
    val zero = (mzero[A], mzero[B])
  }
```

But is there such a thing as a monoid _coproduct_? Could we just use `Either[A,B]` for monoids `A` and `B`? What would be the `zero` of such a monoid? And what would be the value of `Left(a) |+| Right(b)`? We could certainly choose an arbitrary rule, and we may even be able to satisfy the monoid laws, but would that mean we have a _monoid coproduct_?

To answer this, we need to know the precise meaning of _product_ and _coproduct_. These come straight from Wikipedia, with a little help from Cale Gibbard.

A _product_ `M` of two monoids `A` and `B` is a monoid such that there exist homomorphisms `fst: M => A`, `snd: M => B`, and for any monoid `Z` and morphisms `f: Z => A` and `g: Z => B` there has to be a unique homomorphism `h: Z => M` such that `fst(h(z)) == f(z)` and `snd(h(z)) == g(z)` for all `z:Z`. In other words, the following diagram must commute:

{% img /images/Product.png %}

A _coproduct_ `W` of two monoids `A` and `B` is the same except the arrows are reversed. It's a monoid such that there exist homomorphisms `left: A => W`, `right: B => W`, and for any monoid `Z` and morphisms `f: A => Z` and `g: B => Z` there has to be a unique homomorphism `h: W => Z` such that `h(left(a)) == f(a)` and `h(right(b)) == g(b)` for all `a:A` and all `b:B`. In other words, the following diagram must commute:

{% img /images/Coproduct.png %}

We can easily show that our `productMonoid` above really is a monoid product. The homomorphisms are the methods `_1` and `_2` on `Tuple2`. They simply map every element of `(A,B)` to a corresponding element in `A` and `B`. The monoid structure is preserved because:

``` scala
(p |+| q)._1 == (p._1 |+| q._1)
(p |+| q)._2 == (p._2 |+| q._2)

zero[(A,B)]._1 == zero[A]
zero[(A,B)]._2 == zero[B]
```

And for any other monoid `Z`, and morphisms `f: Z => A` and `g: Z => B`, we can construct a unique morphism from `Z` to `(A,B)`:

``` scala
def factor[A:Monoid,B:Monoid,Z:Monoid](z: Z)(f: Z => A, g: Z => B): (A, B) =
  (f(z), g(z))
```

And this really is a homomorphism because we just inherit the homomorphism law from `f` and `g`.

What does a coproduct then look like? Well, it's going to be a type `C[A,B]` together with an instance `coproduct[A:Monoid,B:Monoid]:Monoid[C[A,B]]`. It will be equipped with two monoid homomorphisms, `left: A => C[A,B]` and `right: B => C[A,B]` that satisfy the following (according to the monoid homomorphism law):

``` scala
(left(a1) |+| left(a2)) == left(a1 |+| a2)
(right(b1) |+| right(b2)) == right(b1 |+| b2)

left[A,B](mzero[A]) == mzero[C[A,B]]
right[A,B](mzero[B]) == mzero[C[A,B]]
```

And additionally, for any other monoid `Z` and homomorphisms `f: A => Z` and `g: B => Z` we must be able to construct a unique homomorphism from `C[A,B]` to `Z`:

``` scala
def fold[A:Monoid,B:Monoid,Z:Monoid](c: C[A,B])(f: A => Z, g: B => Z): Z
```

Right off the bat, we know some things that _definitely won't work_. Just using `Either` is a non-starter because there's no well-defined `zero` for it, and there's no way of appending a `Left` to a `Right`. But what if we just added that structure?

### Free monoids on coproducts ###

The underlying set of a monoid `A` is just the type `A` without the monoid structure. The coproduct of types `A` and `B` is the type `Either[A,B]`. Having "forgotten" the monoid structure of both `A` and `B`, we can recover it by generating a free monoid on `Either[A,B]`, which is just `List[Either[A,B]]`. The `append` operation of this monoid is list concatenation, and the identity for it is the empty list.

Clearly `List[Either[A,B]]` is a monoid, but does it permit a homomorphism from both monoids `A` and `B`? If so, then the following properties should hold:

``` scala
List(Left(a1)) ++ List(Left(a2))) == List(Left(a1 |+| a2))
List(Right(b1)) ++ List(Right(b2))) == List(Right(b1 |+| b2))
```

They clearly do not hold! The lists on the left of `==` will have two elements and the lists on the right will have one element. Can we do something about this?

Well, the fact is that `List[Either[A,B]]` is not exactly the monoid coproduct of `A` and `B`. It's still "too big". The problem is that we can observe the internal structure of expressions.

What we need is not exactly the `List` monoid, but a new monoid called the _free monoid product_:

``` scala
sealed class Eithers[A:Monoid,B:Monoid](
  private val toList: List[Either[A,B]]) {

    def ++(p: Eithers[A,B]): Eithers[A,B] =
      Eithers(toList ++ p.toList)

    def fold[Z:Monoid](f: A => Z, g: B => Z): Z =
      toList.foldRight(mzero[Z]) {
        case (Left(a), z) => f(a) |+| z
        case (Right(b), z) => g(b) |+| z
      }
}

object Eithers {
  def left[A:Monoid,B:Monoid](a: A): Eithers[A,B] =
    Eithers(List(Left(a)))
  def right[A:Monoid,B:Monoid](b: B): Eithers[A,B] =
    Eithers(List(Right(b)))

  def empty[A:Monoid,B:Monoid]: Eithers[A,B] = new Eithers(Nil)
  
  def apply[A:Monoid,B:Monoid](xs: List[Either[A,B]]): Eithers[A,B] =
    new Eithers(xs.foldRight(List[Either[A,B]]()) {
      case (Left(a1), Left(a2) :: xs) => Left(a1 |+| a2) :: xs
      case (Right(b1), Right(b2) :: xs) => Right(b1 |+| b2) :: xs
      case (e, xs) => e :: xs
    })
}
```

`Eithers[A,B]` is a kind of `List[Either[A,B]]` that has been normalized so that consecutive `A`s and consecutive `B`s have been collapsed using their respective monoids. So it will contain alternating `A` and `B` values.

The only remaining problem is that a list full of identities is not exactly the same as the empty list. Remember the unit part of the homomorphism law:

``` scala
Eithers(List(Left(mzero))) == Eithers.empty
Eithers(List(Right(mzero))) == Eithers.empty
```

This doesn't hold at the moment. As Cale Gibbard points out in the comments below, `Eithers` is really the free monoid on the coproduct of _semigroups_ `A` and `B`.

We could check each element as part of the normalization step to see if it `equals(zero)` for the given monoid. But that's a problem, as there are lots of monoids for which we can't write an `equals` method. For example, for the `Int => Int` monoid (with composition), we must make use of a notion like extensional equality, which we can't reasonably write in Scala.

So what we have to do is sort of wave our hands and say that equality on `Eithers[A,B]` is defined as whatever notion of equality we have for `A` and `B` respectively, with the rule that `es.fold[(A,B)]` defines the equality of `Eithers[A,B]`. For example, for monoids that really can have `Equal` instances:

``` scala
def eithersEqual[A:Monoid:Equal,B:Monoid:Equal] =
  new Equal[Eithers[A,B]] {
    def equal(xs: Eithers[A,B], ys: Eithers[A,B]) = {
      def le(a: A): (A,B) = (a, mzero[B])
      def ri(b: B): (A,B) = (mzero[A], b)
      def z(es: Eithers[A,B]): (A,B) =
        es.fold(le, ri)(Monoid[(A,B)])
      
      z(xs) === z(ys)
    }
  }
```

So we have to settle with a list full of zeroes being "morally equivalent" to an empty list. The difference is observable in e.g. the time it takes to traverse the list.

Setting that issue aside, `Eithers` is a monoid coproduct because it permits monoid homomorphisms from `A` and `B`:

``` scala
Eithers(List(Left(a1))) ++ Eithers(List(Left(a2)))) ==
  Eithers(List(Left(a1 |+| a2)))
Eithers(List(Right(b1))) ++ Eithers(List(Right(b2)))) ==
  Eithers(List(Right(b1 |+| b2)))
```

And `fold` really is a homomorphism, and we can prove it by case analysis. Here's the law again:

``` scala
(e1.fold(f,g) |+| e2.fold(f,g)) == (e1 ++ e2).fold(f,g)
```

If either of `e1` or `e2` is empty then the result is the fold of the other, so those cases are trivial. If they are both nonempty, then they will have one of these forms:

``` scala
e1 = [..., Left(a1)]
e2 = [Left(a2), ...]

e1 = [..., Right(b1)]
e2 = [Right(b2), ...]

e1 = [..., Left(a)]
e2 = [Right(b), ...]

e1 = [..., Right(b)]
e2 = [Left(a), ...]
```

In the first two cases, on the right of the `==` sign in the law, we perform `a1 |+| a2` and `b1 |+| b2` respectively before concatenating. In the other two cases we simply concatenate the lists. The `++` method on `Eithers` takes care of doing this correctly for us. On the left of the `==` sign we fold the lists individually and they will be alternating applications of `f` and `g`. So then this law amounts to the fact that `f(a1 |+| a2) == f(a1) |+| f(a2)` in the first case, and the same for `g` in the second case. In the latter two cases this amounts to a homomorphism on `List`. So as long as `f` and `g` are homomorphisms, so is `_.fold(f,g)`. Therefore, `Eithers[A,B]` is a coproduct of `A` and `B`.

