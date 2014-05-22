---
layout: post
title: "A Critique of Impure Reason"
date: 2009-04-27
comments: true
categories: 
author: Rúnar
commentIssueId: 9
---

Much has been made of the difference between "object-oriented" and "functional" programming. As much as I've seen people talk about this difference, or some kind of imperative/functional dichotomy, I don't think it's a meaningful distinction. I find that the question of OO vs FP, or imperative vs functional, is not interesting or relevant. The relevant question is one of purity, or the distinction between pure and impure programming. That is, respectively, the kind of programming that eschews data mutation and side-effects as against the kind that embraces them.

Looking at pure (referentially transparent, purely functional) programming from a perspective of having accepted the premise of impurity, it's easy to be intimidated, and to see it as something foreign. Dijkstra once said that some programmers are "mentally mutilated beyond hope of regeneration". I think that on one account, he was right, and that he was talking about this mental barrier of making the leap, from thinking about programming in terms of data mutation and effects, to the "pure" perspective. But I believe he was wrong on the other account. There is hope.

Contrary to what the last post indicated, this will not be a post about problems in "object-oriented" programming in particular, but about problems in impure programming in general. I will illustrate the issue of data mutation by way of two difficulties with it:

* It blurs the distinction between values and variables, such that a term can be both a value and a variable.
* It introduces an artificial distinction between equality and identity.

I will talk briefly about what the implications of each are, and what the alternative view is. But first I will discuss what I think motivates the impure perspective: an aversion to abstraction.

## The Primacy of the Machine ##

In the early days of programming, there were no computers. The first programs were written, and executed, on paper. It wasn't until later that machines were first built that could execute programs automatically.

During the ascent of computers, an industry of professional computer programmers emerged. Perhaps because early computers were awkward and difficult to use, the focus of these professionals became less thinking about programs and more manipulating the machine.

Indeed, if you read the Wikipedia entry on "Computer Program", it tells you that computer programs are "instructions for a computer", and that "a computer requires programs to function". This is a curious position, since it's completely backwards. It implies that programming is done in order to make computers do things, as a primary. I'll warrant that the article was probably written by a professional programmer.

But why does a computer need to function? Why does a computer even exist? The reality is that computers exist solely for the purpose of executing programs. The machine is not a metaphysical primary. Reality has primacy, a program is a description, an abstraction, a proof of some hypothesis about an aspect of reality, and the computer exists to deduce the implications of that fact for the pursuit of human values.

## Shaping Programs in the Machine's Image ##

There's a certain kind of programming that lends itself well to manipulating a computer at a low level. You must think in terms of copying data between registers and memory, and executing instructions based on the machine's current state. In this kind of activity, the programmer serves as the interpreter between the abstract algorithm and the physical machine. But this in itself is a mechanical task. We can instead write a program, once and for all, that performs the interpretation for us, and direct that program using a general-purpose programming language.

But what kind of structure will that language have? Ideally, we would be able to express programs in a natural and concise notation without regard to the machine. Such was the motivation behind LISP. It was designed as an implementation of the lambda calculus, a minimal formal system for expressing algorithms.

This is not what has happened with languages like Java, C++, and some other of today's most popular languages. The abstractions in those languages largely simulate the machine. You have mutable records, or objects, into which you copy data and execute instructions based on their current state. The identity of objects is based on their location in the computer's memory. You have constructs like "factories", "builders", "generators"... _machines_!

What we have is a generation of programmers accustomed to writing programs as if they were constructing a machine. We hear about "engineering" and "architecture" when talking about software. Indeed, as users, we interact with software using pictures of buttons, knobs, and switches. More machinery.

## Equality and Identity ##

Programmers are generally used to thinking of equality and identity as distinct. For example, in Java, `(x == y)` means that the variable `x` holds a pointer to the same memory address as the variable `y`, i.e. that the variable `x` is identical with `y`. However, by convention, `x.equals(y)` means that the object at the memory address pointed to by `x` is equivalent in some sense to the object at the memory address pointed to by `y`, but `x` and `y` are not necessarily identical.

Note that the difference between the two can only be explained in terms of the underlying machine. We have to invoke the notion of memory addresses to understand what's going on, or else resort to an appeal to intuition with terms like "same object", "same instance", etc.

In a pure program, side-effects are disallowed. So a function is not allowed to mutate arguments or make calls to functions that have side-effects. So the result of every function is purely determined by the value of its arguments, and a function does nothing except compute a result. Think about the implications of that for a moment. Given that constraint, is there any reason to maintain a distinction between equality and identity? Consider this example:

{% codeblock lang:java %}

String x = new String("");
String y = new String("");

{% endcodeblock %}

If the values referenced by `x` and `y` are forever immutable, what could it possibly mean for them to not be identical? In what meaningful sense are they not the "same object"?

Note that there's a philosophical connection here. The notion that every object carries an extradimensional identity that is orthogonal to its attributes is a very Platonic idea. The idea that an object has, in addition to its measurable attributes, a distinguished, eternal, noumenal identity. Contrast this with the Aristotelian view, in which an object's identity is precisely the sum of its attributes.

## Variables and Pseudovariables ##

So we find that, given the mutability premise, a term in the language may be not be equal to itself even as it remains identical with itself. In a sense, this is because it's not entirely clear whether we are comparing values or variables when we compare a term. If that sounds a bit strange, consider the following snippet:

{% codeblock lang:java %}
public class Foo {
  public int v;
  public Foo(int i) {
    v = i;
  }
}
...
public Foo f(Foo x) {
  x.v = x.v + 1;
  return x;
}
...
Foo x = new Foo(0);
boolean a = f(x) == f(x); // true
boolean b = f(x).v > f(x).v; // ???
{% endcodeblock %}

Even that short and ostensibly simple code snippet is difficult to comprehend properly, because the `f` method mutates its argument. It seems that since the first boolean is true, the same object appears on either side of the `==` operator, so `f(x)` is identical with `f(x)`. They're the "same object". Even so, it is not clear that this "same object" is equal to itself. To figure out the value of `b`, you have to know some things about how the machine executes the program, not just what the program actually says. You can argue about pass-by-reference and pass-by-value, but these are terms that describe the machine, not the program.

You will note that the `Foo` type above is mutable in the sense that it has accessible components that can be assigned new values. But what does that mean? Is an object of type `Foo` a value or a variable? It seems to _have_ a kind of pseudo-variable `v`. To clarify this, we can imagine that a variable holding a value of type `Foo` also holds a further value of type `int`. Assignment to a variable `x` of type `Foo` also assigns the variable `x.v` of type `int`, but the variable `x.v` can be assigned independently of `x`.

So data mutation is, ultimately, variable assignment. Above, `Foo` is mutated by assigning a value to the pseudo-variable `v`.

Now, I say pseudo-variable, because it's not obvious whether this is a value or a variable. A function that receives a value of type `Foo`, also receives the variable `Foo.v`, to which it can assign new values. So the variable is passed as if it were a value. Stated in terms of the machine, a pointer to a pointer is being passed, and the memory at that address can be mutated by writing another pointer to it. In terms of the program, the called function is able to _rebind a variable in the scope of the caller_.

This is why you have bugs, why programs are hard to debug, and software is difficult to extend. It's because programs are difficult to comprehend to the extent that they are impure. And the reason for that is that they are not sufficiently abstract. A writer of impure code is not operating on the level of abstraction of programming. He's operating at the level of abstraction of the machine.

## Thinking Pure Thoughts ##

In a pure program, there are no side-effects at all. Every function is just that, a function from its arguments to its result. Values are immutable and eternal. A is A, and will always remain A. The only way to get from A to B is for there to be a function that takes A and returns B.

To a lot of programmers used to thinking in terms of mutable state, this perspective is completely baffling. And yet, most of them use immutable values all the time, without finding them particularly profound. For example:

{% codeblock lang:java %}
int x = 1;
int y = x;
int z = x + 1;
{% endcodeblock %}

In that example, 1 is an immutable value. The + operator doesn't modify the value of 1, nor of x, nor of y. It doesn't take the value 1 and turn it into a 2. Every integer is equal to and identical with itself, always and forever.

But what's special about integers? When adding a value to a list, a Java programmer will ordinarily do something like this:

{% codeblock lang:java %}list.add(one);{% endcodeblock %}

This modifies the list in place, so that any variable referencing it will change values. But why shouldn't lists be as immutable as integers? I'm sure some readers just mumbled something about primitives and passing by reference versus passing by value. But why the distinction? If everything were immutable, you would never know if it were passed by value or reference under the covers. Nor would you care what's primitive and what isn't. For an immutable list, for example, the add function would not be able to modify the list in place, and so would return a new one:

{% codeblock lang:java %}list = list.add(one);{% endcodeblock %}

From this perspective, you can write programs that perform arithmetic with complex values as if they were primitive. But why stop with lists? When functions are pure, we can understand programs themselves as objects that are subject to calculation just like numbers in arithmetic, using simple and composable operators.

## Assignment in a Pure World ##

Let's take a simple example of just such an operator. We've talked about how mutation is really assignment. Now let's take the leap fully into the pure perspective, in which assignment is really just closure (pure functions with free variables). To illustrate, here is an example that all Java and C programmers are familiar with:

{% codeblock lang:java %}
int x = 1;
int y = x;
x = y + 1;
t(x + y)
{% endcodeblock %}

The familiar semicolon represents a combinator called "bind". In Haskell, this operator is denoted `(>>=)`. In Scala, it's called `flatMap`. To better see how this works, here's a rough transliteration into Haskell:

{% codeblock lang:haskell %}
f 1
  where
    f x = return x >>= g
    g y = return (y + 1) >>= \x -> t (x + y)
{% endcodeblock %}

Of course, this isn't at all the kind of code you would normally write, but it serves to demonstrate the idea that assignment can be understood as binding the argument to a function. The Haskell code above roughly means:

_Pass `1` to `f`, where `f` passes its argument to unchanged to `g`. And `g` adds `1` to its argument and passes the result to a function that adds `y` to its argument and passes that to `t`._

So in the Java version, we can understand everything after the first semicolon to be a _function_ that takes an argument `x` and we happen to be passing it `1`. Everything after the second semicolon is a function that takes an argument `y` and we happen to be passing it `x` (which has the value `1`). And everything after the third semicolon is a function that takes an argument called `x` (shadowing the first `x`) and passes `x + y` to some function `t`.

Haskell is a purely functional language, and purity is enforced by its type system. If you haven't taken it for a spin, I challenge you to do so. For more on the benefits of purely functional programming, see "Why Haskell Matters".

Of course, purity is not purely the prerogative of Haskell. Sure, Haskell enforces purity, but you can write pure code in any language. That Java code above is pure, you know. If this is all new to you, I challenge you to give purity a shot. In your favourite language, be it Java or something else, start using immutable datastructure in preference to mutable ones. Try thinking in terms of functions from arguments to results rather than in terms of mutation and effects. For Java, you might consider using the immutable datastructures provided by Functional Java or Google Collections. Scala also has immutable datastructures as part of its standard libraries.

It's time that we graduate from punching buttons and pulling levers — manipulating the machines. Here's to programming as it could be and ought to be.

