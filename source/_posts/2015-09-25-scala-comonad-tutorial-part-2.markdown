---
layout: post
title: "Scala Comonad Tutorial, Part 2"
date: 2015-09-25 22:36:31 +0100
comments: true
published: false
author: Rúnar
categories: 
---

In the [previous post](2015-06-23-a-scala-comonad-tutorial.html), we looked at the Reader/Writer monads and comonads, and discussed in general what comonads are. Let's now look at some more comonads

We also considered the question of how e.g. the `Reader` monad and the `Coreader` comonad are related.

Before we get to that, let's look at a few more examples of comonads to get a better feel for how they work. We've seen the `Coreader` and `Cowriter` comonads that let us compose operations that read from a context and log to a monoid, respectively.

The 

## The relationship between Reader and Coreader ##

If we look at a kleisli arrow in the `Reader[R,?]` comonad, it looks like `A => Reader[R,B]`, or expanded out: `A => R => B`. If we uncurry that, we get `(A, R) => B`, and we can go back to the original by currying again. But notice that a value of type `(A, R) => B` is a cokleisli arrow in the `Coreader` comonad! Remember that `Coreader[R,A]` is really a pair `(A, R)`.

So the answer to the question of how `Reader` and `Coreader` are related is that _their arrows are isomorphic_ to one another. There is a one-to-one correspondence between a Kleisli arrow in the `Reader` monad and a Cokleisli arrow in the `Coreader` comonad. This isomorphism is witnessed by currying and uncurrying.

In general, if we have an isomorphism between arrows like this, we have what's called an _adjunction_.


