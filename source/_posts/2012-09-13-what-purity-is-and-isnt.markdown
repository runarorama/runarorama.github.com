---
layout: post
title: "What purity is and isn't"
date: 2012-09-13 22:07
comments: true
published: true
commentIssueId: 3
---

A lot of discussion about "purity" goes on without participants necessarily having a clear idea of what it means exactly. Such discussion is generally unhelpful and distracting.

### What purity is ###

The typical definition of purity (and the one we use in [our book](http://manning.com/bjarnason)) goes something like this:

An expression `e` is _referentially transparent_ if for all programs `p`, every occurrence of `e` in `p` can be replaced with the result of evaluating `e` without changing the result of evaluating `p`.

A function `f` is _pure_ if the expression `f(x)` is referentially transparent for all referentially transparent `x`.

Now, something needs to be made clear right up front. Like all definitions, this holds in a specific _context_. In particular, the context needs to specify what "evaluating" means. It also needs to define "program", "occurrence", and the semantics of "replacing" one thing with another.

In a programming language like Haskell, Java, or Scala, this context is pretty well established. The process of evaluation is a reduction to some _normal form_ such as weak head or beta normal form.

### A simple example ###

To illustrate, let's consider programs in an exceedingly simple language that we will call _Sigma_. An expression in Sigma has one of the following forms:

* A literal character string like `"a"`, `"foo"`, `""`, etc.
* A concatenation, `s + t`, for expressions `s` and `t`.
* A special `Ext` expression that denotes input from an external source.

Now, without an evaluator for Sigma, it is a purely abstract algebra. So let's define a straigtforward evaluator `eval` for it, with the following rules:

* A literal string is already in normal form.
* `eval(s + t)` first evaluates `s` and `t` and concatenates the results into one literal string.
* `eval(Ext)` reads a line from standard input and returns it as a literal string.

This might seem very simple, but it is still not clear whether `Ext` is referentially transparent with regard to `eval`. It depends. What does "reads a line" mean, and what is "standard input" exactly? This is all part of a context that needs to be established.

Here's one implementation of an evaluator for Sigma, in Scala:

{% codeblock lang:scala %}
sealed trait Sigma
case class Lit(s: String) extends Sigma
case class Concat(s: Sigma, t: Sigma) extends Sigma
case object Ext extends Sigma

def eval1(sig: Sigma): Sigma = sig match {
  case Concat(s, t) => {
    val Lit(e1) = eval1(s)
    val Lit(e2) = eval1(t)
    Lit(e1 + e2)
  }
  case Ext => Lit(readLine)
  case x => x
}
{% endcodeblock %}

Now, it's easy to see that the `Ext` instruction is _not_ referentially transparent with regard to `eval1`. Replacing `Ext` with `eval1(ext)` does not preserve meaning. Consider this:

{% codeblock lang:scala %}
val x = Ext
eval1(Concat(x, x))
{% endcodeblock %}

VS this:

{% codeblock lang:scala %}
val x = eval1(Ext)
eval1(Concat(x, x))
{% endcodeblock %}

That's clearly not the same thing. The former will get two strings from standard input and concatenate them together. The latter will get only one string, store it as `x`, and return `x + x`.

Now consider a slightly different evaluator:

{% codeblock lang:scala %}
def eval2(sig: Sigma, stdin: String): Sigma = sig match {
  case Concat(s, t) => {
    val Lit(e1) = eval2(s, stdin)
    val Lit(e2) = eval2(t, stdin)
    Lit(e1 + e2)
  }
  case Ext => Lit(stdin)
  case x => x
}
{% endcodeblock %}

In this case, the `Ext` instruction clearly _is_ referentially transparent with regard to `eval2`, because our standard input is just a string, and it is always the same string. So you see, the purity of functions in the Sigma language very much depends on how that language is interpreted.

This is the reason why Haskell programs are considered "pure", even in the presence of `IO`. A value of type `IO a` in Haskell is simply a function. Reducing it to normal form (evaluating it) has no effect. An `IO` action is of course not referentially transparent with regard to `unsafePerformIO`, but as long as your program does not use that it remains a referentially transparent expression.


### What purity is not ###

In my experience there are more or less two camps into which unhelpful views on purity fall.

The first view, which we will call the _empiricist_ view, is typically taken by people who understand "pure" as a pretentious term, meant to denegrate regular everyday programming as being somehow "impure" or "unclean". They see purity as being "academic", detached from reality, in an ivory tower, or the like.

This view is premised on a superficial understanding of purity. The assumption is that purity is somehow about the absence of I/O, or not mutating memory. But how could any programs be written that don't change the state of memory? At the end of the day, you have to update the CPU's registers, write to memory, and produce output on a display. A program has to make the computer _do something_, right? So aren't we just pretending that our programs don't run on real computers? Isn't it all just an academic exercise in making the CPU warm?

Well, no. That's not what purity means. Purity is not about the absence of program behaviors like I/O or mutable memory. It's about delimiting such behavior in a specific way.

The other view, which I will call the _rationalist_ view, is typically taken by people with overexposure to modern analytic philosophy. Expressions are to be understood by their _denotation_, not by reference to any evaluator. Then of course every expression is really referentially transparent, and so purity is a distinction without a difference. After all, an imperative side-effectful C program can have the same denotation as a monadic, side-effect-free Haskell program. There is nothing wrong with this viewpoint, but it's not instructive _in this context_. Sure, when designing in the abstract, we can think denotationally without regard to evaluation. But when concretizing the design in terms of an actual programming language, we do need to be aware of how we expect evaluation to take place. And only then are referential transparency and purity useful concepts.

Both of these, the rationalist and empiricist views, conflate different levels of abstraction. A program written in a programming language is not the same thing as the physical machine that it may run on. Nor is it the same thing as the abstractions that capture its meaning.

### Further reading ###

I highly recommend the paper [_What is a Purely Functional Language?_](http://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&cad=rja&sqi=2&ved=0CCMQFjAB&url=http%3A%2F%2Fwww.cs.indiana.edu%2F~sabry%2Fpapers%2FpurelyFunctional.ps&ei=I5NYUIRUqvXSAbC_gKgF&usg=AFQjCNGwxjzB5zUBws6D9wnKPzo-zL57pw&sig2=HgE-ZhzoIS19TEe2K2EW-Q) by Amr Sabry, although it deals with the idea of a _purely functional language_ rather than purity of functions within a language that does not meet that criteria.

