---
layout: post
title: "Category Theory, part 1: motivation, definition, examples"
date: 2016-03-17 21:09:31 -0400
comments: true
categories: 
---

[At work](http://takt.com), we're currently doing a category theory reading group, where we're going through Steve Awodey's excellent book [Category Theory](https://global.oup.com/academic/product/category-theory-9780199237180?cc=us&lang=en&). The book is very dense, so the plan is to go through it quite slowly, lingering on each chapter for potentially several weeks.

I make a lot of notes while reading the material, and I'm going to experiment with posting those notes here. These notes assume that the reader has read the relevant section of Awodey's book.

This first part consists of notes on sections `1.1` through `1.4` in chapter 1.

## Why should we care about category theory?

As programmers, we're pretty comfortable with abstraction. Whenever we see a repeated pattern in our programs, we like to capture the idea of the pattern in an abstract interface, or reusable function. Abstraction lets us leave out irrelevant details and talk about only (and exactly) the idea we're interested in.

But even in this process of abstraction, there are patterns. What if we could capture those patterns in interfaces, and have a _framework for abstraction_ itself? This is what category theory gives us. It lets us talk about programs in a very abstract and precise way.

And since the languages of mathematics, physics, and other disciplines can also be phrased in terms of category theory, we can make use of insights from these fields. We can draw on literally thousands of years of human thought to help us write programs and understand software systems.

## What is category theory exactly?

Category theory is the study of _categories_. To begin, let's look at a category that we're all familiar with as programmers: _a category of programs_.

### The category of types and functions

Each expression or value in a programming language like Scala, or Java, or Haskell, has a _type_. We have types like `Int`, `Bool`, `[String]`, and the like.

More importantly, we have _functions_. A function has a type like `a -> b`, which means it takes a value of type `a` and computes a value of type `b`.

Functions also _compose_. That is, for any two functions `f: b -> c` and `g: a -> b`, we can get a composite function `f . g : a -> c`.

``` haskell
(.) :: (b -> c) -> (a -> b) -> a -> c
(f . g) a = f (g a)
```

Composition of functions is _associative_. That is, for any three functions `f`, `g`, and `h`:

``` haskell
f . (g . h) = (f . g) . h
```

Either way, we get the composite function `λa -> f (g (h a))`.

For every type `a` there's an identity function `id: a -> a` that "does nothing":

``` haskell 
id :: a -> a
id a = a
```

This function is a "unit" for composition:

``` scala
f . id = f
id . f = f
```

Because `f (id a)` is just `f a` and `id (g b)` is just `g b`.

As Awodey says, if we "abstract away" everything about functions, except composition and identities, we get an abstract notion called a _category_.

### What is a category exactly?

A category is something incredibly simple. It just consists of 2 things:

1. Some objects (call these `A`, `B`, `C`, etc).
2. Some arrows (call these `f`, `g`, `h`, etc) between objects. We will write `f: A -> B` to denote that the arrow `f` points from the object `A` to the object `B`.

It's important to note that these are not necessarily _sets_ of objects and arrows. They could be, but they don't have to be. Also, what an "object" is exactly is not defined here. That's an implementation detail, and each category's objects will be different. The abstract interface of a category stipulates nothing about the objects themselves.

Although there are no rules with regard to the objects, there are some rules regarding the arrows:

1. For any two arrows `g: A -> B` and `f: B -> C`, there is a composite arrow `f . g` which goes from `A` to `C`. (It's good to read `f . g` as "`f` after `g`" or even "`f` of `g`").
2. Composition of arrows is associative. That is, it doesn't matter if we group `f . g . h` as `f . (g . h)` or `(f . g) . h`.
3. For every object `A` there is an identity arrow `id[A]: A -> A`. This arrow is a unit with regard to composition. That is, `f . id = f` and `id . f = f` for any arrow `f`.

And that's it. You now know essentially all of category theory. It's deceptively simple. But there are lots of questions to explore. For instance:

* What are some examples of categories?
* What kinds of things can we construct from categories?
* Can we compare one category with another?
* How does this help us with day-to-day programming?

## Examples of categories

We've already seen the category of types and functions. The objects of this category are types, and the arrows are functions. A function of type `a -> b` is an arrow from the type `a` to the type `b`. Composition of arrows is just function composition, and the `id` function implements all of the identity arrows.

Since this category is our first and most familiar example, it can be tempting to always think of arrows as being functions or "something like functions". But some categories have arrows that are wildly different from functions, as we shall see.

### Preorder categories

A preorder is just a set equipped with a binary relation `≤` that is transitive and reflexive. For example, the integers form a preorder such that for any two integers `a` and `b`, we have `a ≤ b` when `a` is less than or equal to `b`. Transitivity means that if `a ≤ b` and `b ≤ c`, then `a ≤ c`. Reflexivity means that `a ≤ a`.

Types in a language with subtyping (like Scala) also form a preorder, where the binary relation is the subtype relationship. For example, in Scala we have `A <: B` when the type `A` is a subtype of `B`. That is, a value of type `A` can be used wherever a value of type `B` is expected.

Any such preorder turns out to form a category. Let's take category of Scala types with subtyping to see how this works. That category is defined as follows:

  * The objects are Scala types `A`, `B`, `C`, etc.
  * There is an arrow `A <: B` from `A` to `B` precisely when `A` is a subtype of `B`. Note that this arrow is not a function. The arrow consists of the subtype relationship and nothing else.
  * Arrows compose via transitivity of subtyping. Given `B <: C` (an arrow from `B` to `C`) and given `A <: B`, there's a composite arrow `A <: C`.
  * For every type `A` there's an identity arrow `A <: A` because every type is a subtype of itself.

### Monoids

A monoid is a set `m` closed under an associative binary operation `mappend :: m -> m -> m`. And there's an identity element `mempty :: m` such that:

``` haskell
mappend mempty x == x
mappend x mempty == x
```

Monoids are everywhere in programming. For example:

* The integers under addition form a monoid with an identity element of `0`.
* So do the integers under multiplication, where the identity element is `1`.
* Strings with concatenation form a monoid, with the empty string as the identity.
* The set of Boolean values `True` and `False` forms a monoid in two ways. One is `&&` with `True`, the other is `||` with `False`.
* For any type `a`, the type of functions `a -> a` (where the domain and codomain are the same) forms a monoid. The `mappend` operation is function composition and the identity element is the identity function.

More abstractly (and more precisely), _a monoid is a category with one object_. We can verify that this works for any monoid `m`. As a category:

  * The only object is the type `m`.
  * Every value `e` of type `m` is an arrow from `m` to `m`.
  * Composition of arrows is `mappend` (sometimes written `<>` in Haskell).
  * The identity arrow is `mempty`.

It's important to note that just like in the preorder category above, the arrows in this category are _not functions_. They are values of type `m`.

The fact that when we regard them as arrows they all go from `m` to `m` simply means that _every arrow composes with every other_. So "category with one object" just means "category where all the arrows are compatible".

### The category of monoids

Not only is each monoid a category in itself, the monoids together form a category called `Mon`.

  * The objects are monoids.
  * The arrows are _monoid homomorphisms_.
  
A monoid homomorphism `h: m -> n` is a structure-preserving mapping in that `h a <> h b = h (a <> b)` and `h (mempty : m) = (mempty : n)`. In other words, a monoid homomorphism preserves composition and identities.

For example, a function that takes the length of a string is a homomorphism, since `s.length + t.length = (s ++ t).length`. Adding the lengths of two strings is the same as taking the length of their concatenation, and the length of the empty string (the identity for concatenation) is zero (the identity for integer addition).

### A category of categories

Since a monoid is a kind of category, and we have a category of monoids, we actually have a category full of categories. We can drop the restriction that the categories in our category need to be monoids, and make the category `Cat` of categories:

  * The objects in this category are categories
  * The arrows are _functors_. A functor `F: C -> D` is a homomorphism from the category `C` to the category `D` in that it preserves the structure of the category:

  1. It takes an arrow `f: a -> b` in `C` to an arrow `F(f): F(a) -> F(b)` in `D`.
  2. `F(id[a]) = id[F(a)]`, meaning it takes the identities in `C` to identities in `D`.
  3. `F(f . g) = F(f) . F(g)`, meaning it takes composite arrows in `C` to composite arrows in `D`.

For example if both categories are the category `Hask` of Haskell types, then a functor from `Hask` to `Hask` is an instance of the `Functor` class in Haskell:

```
class Functor f where
  fmap :: (a -> b) -> f a -> f b
```

Note that for this to really be a functor, it has to satisfy a homomorphism law:

1. `fmap id = id`
2. `fmap (f . g) = fmap f . fmap g`

A functor from a category to itself is called an _endofunctor_ (endo meaning "within").

### A category of endofunctors

For any given category `C`, there's a whole category of endofunctors in `C`.

  * The objects are endofunctors in `C`.
  * The arrows are homomorphisms among functors. These are called _natural transformations_.

As you might expect, a natural transformation `h: F -> G` (transforming the functor `F` to the functor `G`) obeys a homomorphism law. For any arrow `c` in the category `C`:

```
h . F(c) = G(c) . h
```

For example, in the category of endofunctors in Haskell, (where objects are instances of the `Functor` class), the natural transformations from a functor `f` to a functor `g` are _polymorphic functions_ of type `forall a. f a -> g a`. The homomorphism law stated in Haskell:

```
h . fmap c = fmap c . h
```

### Finite categories

We can construct very simple categories from scratch. For example, there's a category called `1` with just one object and one arrow, the identity arrow on that object. Note that this category is a monoid. What the object in this category is doesn't matter. It can be anything we want.

There's also `2` with two objects and a single arrow from one to the other (as well as two identity arrows). Again, it doesn't matter what these two objects are or what the arrow between them represents.

An even simpler specimen is the empty category `0`. It has no objects at all, and therefore no arrows either.


