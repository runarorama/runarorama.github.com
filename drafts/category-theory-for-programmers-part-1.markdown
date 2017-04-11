---
layout: post
title: "Category theory in everyday programming, part 1"
date: 2016-03-17 21:09:31 -0400
comments: true
categories: 
---

This post is the first in what's going to be a series on category theory, from the perspective of day-to-day software development. I'm going to give code examples in Scala, mainly because that will allow me to refer to my book _Functional Programming in Scala_ and use it as a resource.

## Why should programmers care about category theory?

As programmers, we're really comfortable with abstraction. Whenever we see a repeated pattern in our programs, we like to capture the idea of the pattern in an abstract interface, template, or framework. This lets us leave out implementation details that are irrelevant to the essence of the idea.

But even in this process of abstraction, there are patterns. What if we could capture those patterns in interfaces, and have a _framework for abstraction_ itself? This is what category theory gives us. It lets us talk about programming in a very abstract and precise way. A major theme of category theory is that abstraction and precision are the same.

## What is category theory exactly?

Category theory is the study of _categories_. To begin, we're going to look at a category that we're all familiar with as programmers: _a category of programs_.

### The category of types and functions

Each expression or value in a programming language like Scala, or Java, or Haskell, has a _type_. We have types like `Int`, `Bool`, `[String]`, and the like.

More importantly, we have _functions_. A function has a type like `a -> b`, which means it takes a value of type `a` and computes a value of type `b`.

We can imagine drawing a diagram where the types in our language are points, and the functions are arrows between them.

TODO: Insert image of the type category

Functions also _compose_. That is, for any two functions `f :: b -> c` and `g :: a -> b`, we can get a composite function `f . g :: a -> c`.

``` scala
def compose[A,B,C](f: B => C, g: A => B): A => C =
  a => f(g(a))
```

This means that in our diagram, if there's an arrow `f` from `B` to `C`, and an arrow `g` from `A` to `B`, there's also an arrow from `A` to `C` which is just the composite function `f compose g`.

TODO: Draw the diagram for composition.

Composition of functions is _associative_. That is, for any three functions `f`, `g`, and `h`:

``` scala
f compose (g compose h) = (f compose g) compose h
```

Either way, we get a composite function `a => f(g(h(a)))`.

For every type `A` there's an identity function `identity[A]: A => A` that "does nothing":

``` scala
def identity[A](a: A): A = a
```

This function is a "unit" for composition:

``` scala
f compose identity = f
identity compose g = g
```

Because `f(identity(a))` is just `f(a)` and `identity(g(b))` is just `g(b)`.

Types, functions, and function composition together form a _category_.

### What is a category exactly?

A category is something incredibly simple. It just consists of 2 things:

1. Some objects (call these `A`, `B`, `C`, etc).
2. Some arrows (call these `f`, `g`, `h`, etc) between objects. We will write `f: A -> B` to denote that the arrow `f` points from the object `A` to the object `B`.

It's important to note that these are not necessarily _sets_ of objects and arrows. They could be, but they don't have to be. Also, what an "object" is exactly is not defined here. That's an implementation detail, and each category's objects will be different. The abstract interface of a category stipulates nothing about the objects themselves.

Although there are no rules with regard to the objects, there are some rules regarding the arrows:

1. For any two arrows `g: A -> B` and `f: B -> C`, there is a composite arrow `f . g` (it's good to read this as "f after g") which goes from `A` to `C`.
2. Composition of arrows is associative. That is, it doesn't matter if we group `f . g . h` as `f . (g . h)` or `(f . g) . h`.
3. For every object `A` there is an identity arrow `id[A]: A -> A`. This arrow is a unit with regard to composition. That is, `f . id = f` and `id . f = f` for any arrow `f`.

And that's it. You now know essentially all of category theory. It's deceptively simple. But there are lots of questions to explore. For instance:

* What are some examples of categories?
* What kinds of things can we construct from categories?
* Can we compare one category with another?
* How does this help us with day-to-day programming?

## Examples of categories

Obviously types and functions form a category. The objects of this category are types, and the arrows are functions that point from their domains to their codomains. Composition of arrows is just function composition, and the `identity[A]` function serves as the identity arrow for the object (the type) `A`.

### Subcategories of Set

### Types and subtypes

### The opposite category
