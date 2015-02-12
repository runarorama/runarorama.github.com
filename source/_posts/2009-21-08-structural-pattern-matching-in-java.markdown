---
layout: post
title: "Structural pattern matching in Java"
date: 2009-08-21
comments: true
categories: 
author: Rúnar
commentssIssueId: 14
---

(updated for Java 8)

One of the great features of modern programming languages is structural pattern matching on algebraic data types. Once you've used this feature, you don't ever want to program without it. You will find this in languages like Haskell and Scala.

In Scala, algebraic types are provided by case classes. For example:

{% codeblock lang:scala %}
sealed trait Tree
case object Empty extends Tree
case class Leaf(n: Int) extends Tree
case class Node(l: Tree, r: Tree) extends Tree
{% endcodeblock %}

To define operations over this algebraic data type, we use pattern matching on its structure:

{% codeblock lang:scala %}
def depth(t: Tree): Int = t match {
  case Empty => 0
  case Leaf(n) => 1
  case Node(l, r) => 1 + max(depth(l), depth(r))
}
{% endcodeblock %}

When I go back to a programming language like, say, Java, I find myself wanting this feature. Unfortunately, algebraic data types aren't provided in Java. However, a great many hacks have been invented over the years to emulate it, knowingly or not.

### The Ugly: Interpreter and Visitor

What I have used most throughout my career to emulate pattern matching in languages that lack it are a couple of hoary old hacks. These venerable and well respected practises are a pair of design patterns from the GoF book: Interpreter and Visitor.

The Interpreter pattern really does describe an algebraic structure, and it provides a method of reducing (interpreting) the structure. However, there are a couple of problems with it. The interpretation is coupled to the structure, with a "context" passed from term to term, and each term must know how to mutate the context appropriately. That's minus one point for tight coupling, and minus one for relying on mutation.

The Visitor pattern addresses the former of these concerns. Given an algebraic structure, we can define an interface with one "visit" method per type of term, and have each term accept a visitor object that implements this interface, passing it along to the subterms. This decouples the interpretation from the structure, but still relies on mutation. Minus one point for mutation, and minus one for the fact that Visitor is incredibly crufty. For example, to get the depth of our tree structure above, we have to implement a TreeDepthVisitor. A good IDE that generates boilerplate for you is definitely recommended if you take this approach.

On the plus side, both of these patterns provide some enforcement of the exhaustiveness of the pattern match. For example, if you add a new term type, the Interpreter pattern will enforce that you implement the interpretation method. For Visitor, as long as you remember to add a visitation method for the new term type to the visitor interface, you will be forced to update your implementations accordingly.

### The Bad: Instanceof

An obvious approach that's often sneered at is runtime type discovery. A quick and dirty way to match on types is to simply check for the type at runtime and cast:

{% codeblock lang:java %}
public static int depth(Tree t) {
  if (t instanceof Empty)
    return 0;
  if (t instanceof Leaf)
    return 1;
  if (t instanceof Node)
    return 1 + max(depth(((Node) t).left), depth(((Node) t).right));
  throw new RuntimeException("Inexhaustive pattern match on Tree.");
}
{% endcodeblock %}

There are some obvious problems with this approach. For one thing, it bypasses the type system, so you lose any static guarantees that it's correct. And there's no enforcement of the exhaustiveness of the matching. But on the plus side, it's both fast and terse.

### The Good: Functional Style

There are at least two approaches that we can take to approximate pattern matching in Java more closely than the above methods. Both involve utilising parametric polymorphism and functional style. Let's consider them in order of increasing preference, i.e. less preferred method first.

#### Safe and Terse - Disjoint Union Types

The first approach is based on the insight that algebraic data types represent a disjoint union of types. Now, if you've done any amount of programming in Java with generics, you will have come across (or invented) the simple pair type, which is a conjunction of two types:

{% codeblock lang:java %}
public abstract class P2<A, B> {
  public A _1();
  public B _2();
}
{% endcodeblock %}

A value of this type can only be created if you have both a value of type `A` and a value of type `B`. So (conceptually, at least) it has a single constructor that takes two values. The disjunction of two types is a similar idea, except that a value of type `Either<A, B>` can be constructed with either a value of type `A` or a value of type `B`:

{% codeblock lang:java %}
public final class Either<A, B> {
  ...
  public static <A, B> Either<A, B> left(A a) { ... }
  public static <A, B> Either<A, B> right(B a) { ... }
  ...
}
{% endcodeblock %}

Encoded as a disjoint union type, then, our `Tree` data type above is: `Either<Empty, Either<Leaf, Node>>`

Let's see that in context. Here's the code.

{% codeblock lang:java %}
public abstract class Tree {
  // Constructor private so the type is sealed.
  private Tree() {}

  public abstract Either<Empty, Either<Leaf, Node>> toEither();

  public static final class Empty extends Tree {
    public <T> T toEither() {
      return left(this);
    }

    public Empty() {}
  }

  public static final class Leaf extends Tree {
    public final int n;

    public Either<Empty, Either<Leaf, Node>> toEither() {
      return right(Either.<Leaf, Node>left(this));
    }

    public Leaf(int n) { this.n = n; }
  }

  public static final class Node extends Tree {
    public final Tree left;
    public final Tree right;    

    public Either<Empty, Either<Leaf, Node>> toEither() {
      return right(Either.<Leaf, Node>right(this));
    }

    public Node(Tree left, Tree right) {
      this.left = left; this.right = right;
    }
  }
}
{% endcodeblock %}

The neat thing is that `Either<A, B>` can be made to return both `Iterable<A>` and `Iterable<B>` in methods `right()` and `left()`, respectively. One of them will be empty and the other will have exactly one element. So our pattern matching function will look like this:

{% codeblock lang:java %}
public int depth(Tree t) {
  Either<Empty, Either<Leaf, Node>> eln = t.toEither();
  for (Empty e: eln.left())
    return 0;
  for (Either<Leaf, Node> ln: eln.right()) {
    for (leaf: ln.left())
      return 1;
    for (node: ln.right())
      return 1 + max(depth(node.left), depth(node.right));
  }
  throw new RuntimeException("Inexhaustive pattern match on Tree.");
}
{% endcodeblock %}

That's terse and readable, as well as type-safe. The only issue with this is that the exhaustiveness of the patterns is not enforced, so we're still only discovering that error at runtime. So it's not all that much of an improvement over the instanceof approach.

#### Safe and Exhaustive: Church Encoding

Alonzo Church was a pretty cool guy. Having invented the lambda calculus, he discovered that you could encode data in it. We've all heard that every data type can be defined in terms of the kinds of operations that it supports. Well, what Church discovered is much more profound than that. A data type IS a function. In other words, an algebraic data type is not just a structure together with an algebra that collapses the structure. The algebra IS the structure.

Consider the boolean type. It is a disjoint union of True and False. What kinds of operations does this support? Well, you might want to do one thing if it's True, and another if it's False. Just like with our Tree, where we wanted to do one thing if it's a Leaf, and another thing if it's a Node, etc.

But it turns out that the boolean type IS the condition function. Consider the Church encoding of booleans:

{% codeblock %}
true  = λa.λb.a
false = λa.λb.b
{% endcodeblock %}

So a boolean is actually a binary function. Given two terms, a boolean will yield the former term if it's true, and the latter term if it's false. What does this mean for our `Tree` type? It too is a function:

{% codeblock %}
empty = λa.λb.λc.a
leaf  = λa.λb.λc.λx.b x
node  = λa.λb.λc.λl.λr.c l r
{% endcodeblock %}

You can see that this gives you pattern matching for free. The `Tree` type is a function that takes three arguments:

A value to yield if the tree is empty.
A unary function to apply to an integer if it's a leaf.
A binary function to apply to the left and right subtrees if it's a node.
The type of such a function looks like this (Scala notation):

{% codeblock lang:scala %}
T => (Int => T) => (Tree => Tree => T) => T
{% endcodeblock %}

Or equivalently:

{% codeblock lang:scala %}
(Empty => T) => (Leaf => T) => (Node => T) => T
{% endcodeblock %}

Translated to Java, we need this method on Tree:

{% codeblock lang:java %}
public abstract <T> T match(Function<Empty, T> a,
                            Function<Leaf, T> b,
                            Function<Node, T> c);
{% endcodeblock %}

The `Function` interface is in the `java.util` package in Java 8, but you can definitely make it yourself in previous versions:

{% codeblock lang:java %}
public interface Function<A, B> { public B apply(A a); }
{% endcodeblock %}

Now our Tree code looks like this:

{% codeblock lang:java %}
public abstract class Tree {
  // Constructor private so the type is sealed.
  private Tree() {}

  public abstract <T> T match(Function<Empty, T> a,
                              Function<Leaf, T> b,
                              Function<Node, T> c);

  public static final class Empty extends Tree {
    public <T> T match(Function<Empty, T> a,
                       Funciton<Leaf, T> b,
                       Function<Node, T> c) {
      return a.apply(this);
    }

    public Empty() {}
  }

  public static final class Leaf extends Tree {
    public final int n;

    public <T> T match(Function<Empty, T> a,
                       Function<Leaf, T> b,
                       Function<Node, T> c) {
      return b.apply(this);
    }

    public Leaf(int n) { this.n = n; }
  }

  public static final class Node extends Tree {
    public final Tree left;
    public final Tree right;    

    public <T> T match(Function<Empty, T> a,
                       Function<Leaf, T> b,
                       Function<Node, T> c) {
      return c.apply(this);
    }

    public Node(Tree left, Tree right) {
      this.left = left; this.right = right;
    }
  }
}
{% endcodeblock %}

And we can do our pattern matching on the calling side:

{% codeblock lang:java %}
public int depth(Tree t) {
  return t.match((Empty e) -> 0,
                 (Leaf l) -> 1,
                 (Node n) -> 1 + max(depth(n.left), depth(n.right));
}
{% endcodeblock %}

This is almost as terse as the Scala code, and very easy to understand. Everything is checked by the type system, and we are guaranteed that our patterns are exhaustive. This is an ideal solution.

### Conclusion

With some slightly clever use of generics and a little help from our friends Church and Curry, we can indeed emulate structural pattern matching over algebraic data types in Java, to the point where it's almost as nice as a built-in language feature.

So throw away your Visitors and set fire to your GoF book.

