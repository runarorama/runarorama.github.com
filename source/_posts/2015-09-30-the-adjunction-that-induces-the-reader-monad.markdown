---
layout: post
title: "The adjunction that induces the reader monad"
date: 2015-09-30 07:48:13 -0400
author: Rúnar
comments: true
categories: 
commentIssueId: 19
---

In writing up part 2 of my [Scala Comonad Tutorial](http://blog.higher-order.com/blog/2015/06/23/a-scala-comonad-tutorial/), and coming up with my talk for [Scala World](http://scala.world), I idly pondered this question:

{% blockquote %}
If all monads are given by composing adjoint pairs of functors, what adjoint pair of functors forms the `Reader` monad? And if we compose those functors the other way, which comonad do we get?
{% endblockquote %}

Following leads from Shachaf Ben-Kiki on IRC, I thought about whether there was a free-forgetful adjunction given by an isomorphism between Kleisli arrows in the Reader monad on one hand and its algebras on the other. But then what are `(R => ?)` algebras, other than values of type `R`?

Searching around for an answer to _that_, I came across [this Mathematics Stack Exchange question](http://math.stackexchange.com/questions/1274989/what-is-the-eilenberg-moore-category-of-this-diagonal-like-monad), whose answer more or less directly addresses my first question above. The adjunction is a little more difficult to see than I expected, but I've worked through it, and what follows is an outline of how I convinced myself that it works.

## The adjoint pair ##

Let's consider the reader monad `R => ?`, which allows us to read a context of type `R`.

The first category involved is _Set_ (or _Hask_, or _Scala_). This is just the familiar category where the objects are types (`A`,`B`,`C`, etc.) and the arrows are functions.

The other category is _Set/R_, which is the [slice category](http://ncatlab.org/nlab/show/overcategory) of _Set_ over the type `R`. This is a category whose objects are functions to `R`. So an object `x` in this category is given by a type `A` together with a function of type `A => R`. An arrow from `x: A => R` to `y: B => R` is given by a function `f: A => B` such that `y(f(a)) = x(a)` for all `a:A`.

The left adjoint is `R*`, a functor from _Set_ to _Set/R_. This functor sends each type `A` to the function `(p:(R,A)) => p._1`, having type `(R,A) => R`.

``` scala
def rStar[A]: (R,A) => R = _._1
```

The right adjoint is `Π_R`, a functor from _Set/R_ to _Set_. This functor sends each object `q: A => R` in _Set/R_ to a type `P_A` in _Set_. This type `P_A` is actually a dependent type inhabited by arrows `p: R => A` which satisfy the identity `q(p(a)) = a` for all `a:A`. That is, `q` has to be the inverse of `p`.

## Constructing the monad #

The monad is not exactly easy to see, but if everything has gone right, we should get the `R => ?` monad by composing `Π_R` with `R*`.

We start with a type `A`. Then we do `R*`, which gives us the object `rStar[A]` in the slice category, which you will recall is just `_._1` of type `(R,A) => A`. Then we go back to types via `Π_R(rStar[A])` which gives us a dependent type `P_(R,A)` inhabited by functions `p: R => (R,A)`. Now, this looks a lot like an action in the `State` monad. But it's not. These `p` must satisfy the property that `_1` is their inverse. Which means that the `R` they return must be exactly the `R` they were given. So it's like a `State` action that is _read only_. We can therefore simplify this to the ordinary (non-dependent) type `R => A`. And now we have our `Reader` monad.

## Constructing the comonad #

But what about the other way around? What is the comonad constructed by composing `R*` with `Π_R`? Well, since we end up in the slice category, our comonad is actually in that category rather than in _Set_.

We start with an object `q: A => R` in the slice category. Then we go to types by doing `Π_R(q)`. This gives us a dependent type `P_A` which is inhabited by all `p: R => A` such that `q` is their inverse. Then we take `rStar[Π_R(q)]` to go back to the slice category and we find ourselves at an object `f: (R, Π_R(q)) => R`, which you'll recall is implemented as `_1`. As an endofunctor in _Set/R_, `λq. rStar[Π_R(q)]` takes all `q: A => R` to `p: (R, R => A) => R = _._1` such that `p` is only defined on `R => A` arguments whose inverse is `q`.

That is, the counit for this comonad on elements `y: A => R` must be a function `counit: (R, Π_R(y)) => A` such that for any `_1: (R, Π_R(y)) => R`, the property `y compose counit = _1` holds. Note that this means that the `R` returned by `_1` and the `R` returned by `y` must be the same. Recall that `_1` always returns the first element of its argument, and also recall that the functions in `Π_R(y)` must have `y` as their inverse, so they're only defined at the first element of the argument to `_1`. That is `p._2(x)` is only defined when `x = p._1`.

If we try to encode that in Scala (ignoring all the "such that"), we get something like:

``` scala
def counit[A](p: (R, R => A)): A =
  p._2(p._1)
```

This looks a lot like a `counit` for the `Store` comonad! Except what we constructed is not that. Because of the additional requirements imposed by our functors and by the slice category, the second element of `p` can only take an argument that is exactly the first element of `p`. So we can simplify that to `(R, () => A)` or just `(R, A)`. And we now have the familiar `Coreader` comonad.

## But how?

Now, finding the answer on StackExchange seems a litte bit of a _deus ex machina_. If you know of a way to calculate an adjoint pair given the monad that comprises them (or if you know that this can't be done in general), please comment below.

