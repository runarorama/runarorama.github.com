---
layout: post
title: "Maximally Powerful, Minimally Useful"
date: 2014-12-21 23:23
comments: true
commentIssueId: 12
---

It's well known that there is a trade-off in language and systems design between expressiveness and analyzability. That is, the more expressive a language or system is, the less we can reason about it, and vice versa. The more capable the system, the less comprehensible it is.

This principle is very widely applicable, and it's a useful thing to keep in mind when designing languages and libraries. A practical implication of being aware of this principle is that we always make components exactly as expressive as necessary, but no more. This maximizes the ability of any downstream systems to reason about our components. And dually, for things that we receive or consume, we should require exactly as much analytic power as necessary, and no more. That maximizes the expressive freedom of the upstream components.

I find myself thinking about this principle a lot lately, and seeing it more or less everywhere I look. So I'm seeking a more general statement of it, if such a thing is possible. It seems that more generally than issues of expressivity/analyzability, a restriction at one semantic level translates to freedom and power at another semantic level.

What I want to do here is give a whole bunch of examples. Then we'll see if we can come up with an integration for them all. This is all written as an exercise in thinking out loud and is not to be taken very seriously.

## Examples from computer science

### Context-free and regular grammars

In formal language theory, context-free grammars are more expressive than regular grammars. The former can describe strictly more sets of strings than the latter. On the other hand, it's harder to reason about context-free grammars than regular ones. For example, we can decide whether two regular expressions are equal (they describe the same set of strings), but this is undecidable in general for context-free grammars.

### Monads and applicative functors

If we know that an applicative functor is a monad, we gain some expressive power that we don't get with just an applicative functor. Namely, a monad is an applicative functor with an additional capability: monadic join (or "bind", or "flatMap"). That is, context-sensitivity, or the ability to bind variables in monadic expressions.

This power comes at a cost. Whereas we can always compose any two applicatives to form a composite applicative, two monads do not in general compose to form a monad. It may be the case that a given monad composes with any other monad, but we need some additional information about it in order to be able to conclude that it does.

### Actors and futures

Futures have an algebraic theory, so we can reason about them algebraically. Namely, they form an applicative functor which means that two futures `x` and `y` make a composite future that does `x` and `y` in parallel. They also compose sequentially since they form a monad.

Actors on the other hand have no algebraic theory and afford no algebraic reasoning of this sort. They are "fire and forget", so they could potentially do anything at all. This means that actor systems can do strictly more things in more ways than systems composed of futures, but our ability to reason about such systems is drastically diminished.

### Typed and untyped programming

When we have an untyped function, it could receive any type of argument and produce any type of output. The implementation is totally unrestricted, so that gives us a great deal of expressive freedom. Such a function can potentially participate in a lot of different expressions that use the function in different ways.

A function of type `Bool -> Bool` however is highly restricted. Its argument can only be one of two things, and the result can only be one of two things as well. So there are 4 different implementations such a function could possibly have. Therefore this restriction gives us a great deal of analyzability.

For example, since the argument is of type `Bool` and not `Any`, the implementation mostly writes itself. We need to consider only two possibilities. `Bool` (a type of size 2) is fundamentally easier to reason about than `Any` (a type of potentially infinite size). Similarly, any usage of the function is easy to reason about. A caller can be sure not to call it with arguments other than `True` or `False`, and enlist the help of a type system to guarantee that expressions involving the function are meaningful.

### Total functional programming

Programming in non-total languages affords us the power of general recursion and "fast and loose reasoning" where we can transition between valid states through potentially invalid ones. The cost is, of course, the halting problem. But more than that, we can no longer be certain that our programs are _meaningful_, and we lose some algebraic reasoning. For example, consider the following:

```
map (- n) (map (+ n) xs)) == xs
```

This states that adding `n` to every number in a list and then subtracting `n` again should be the identity. But what if `n` actually throws an exception or never halts? In a non-total language, we need some additional information. Namely, we need to know that `n` is total.

### Referential transparency and side effects

The example above also serves to illustrate the trade-off between purely functional and impure programming. If `n` could have arbitrary side effects, algebraic reasoning of this sort involving `n` is totally annihilated. But if we know that `n` is referentially transparent, algebraic reasoning is preserved. The power of side effects comes at the cost of algebraic reasoning. This price includes loss of compositionality, modularity, parallelizability, and parametricity. Our programs can do strictly more things, but we can conclude strictly fewer things about our programs.

## Example from infosec

There is a principle in computer security called _The Principle of Least Privilege_. It says that a user or program should have exactly as much authority as necessary but no more. This constrains the power of the entity, but greatly enhances the power of others to predict and reason about what the entity is going to do, resulting in the following benefits:

- **Compositionality** -- The fewer privileges a component requires, the easier it is to deploy inside a larger environment. For the purposes of safety, higher privileges are a barrier to composition since a composite system requires the _highest_ privileges of any of its components.
- **Modularity** -- A component with restricted privileges is easier to reason about in the sense that its interaction with other components will be limited. We can reason mechanically about where this limit actually is, which gives us better guarantees about the the security and stability of the overall system. A restricted component is also easier to test in isolation, since it can be run inside an overall restricted environment.

## Example from politics

Some might notice an analogy between the Principle of Least Privilege and the idea of a constitutionally limited government. An absolute dictatorship or pure democracy will have absolute power to enact whatever whim strikes the ruler or majority at the moment. But the overall stability, security, and freedom of the people is greatly enhanced by the presence of legal limits on the power of the government. A limited constitutional republic also makes for a better neighbor to other states.

More generally, a ban on the initiation of physical force by one citizen against another, or by the government against citizens, or against other states, makes for a peaceful and prosperous society. The "cost" of such a system is the inability of one person (or even a great number of people) to impose their preferences on others by force.

## An example from mathematics

The framework of two-dimensional Euclidean geometry is simply an empty page on which we can construct lines and curves using tools like a compass and straightedge. When we go from that framework to a Cartesian one, we constrain ourselves to reasoning on a grid of pairs of numbers. This is a tradeoff between expressivity and analyzability. When we move fom Euclidean to Cartesian geometry, we lose the ability to assume isotropy of space, intersection of curves, and compatibility between dimensions. But we gain much more powerful things through the restriction: the ability to precisely define geometric objects, to do arithmetic with them, to generalize to higher dimensions, and to reason with higher abstractions like linear algebra and category theory.

## Examples from everyday life

### Driving on roads

Roads constrain the routes we can take when we drive or walk. We give up moving in a straight line to wherever we want to go. But the benefit is huge. Roads let us get to where we're going much faster and more safely than we would otherwise.

### Commodity components

Let's say you make a decision to have only one kind of outfit that you wear on a daily basis. You just go out and buy multiple identical outfits. Whereas you have lost the ability to express yourself by the things you wear, you have gained a certain ability to reason about your clothing. The system is also fault-tolerant and compositional!

## Summary

What is this principle? Here are some ways of saying it:

* Things that are maximally general for first-order applications are minimally useful for higher-order applications, and vice versa.
* A language that is maximally expressive is minimally analyzable.
* A simplifying assumption at one semantic level paves the way to a richer structure at a higher semantic level.

What do you think? Can you think of a way to integrate these examples into a general principle? Do you have other favorite examples of this principle in action? Is this something everyone already knows about and I'm just late to the party?

