---
layout: post
title: "Easy performance wins with Scalaz"
date: 2015-06-18 15:09:21 -0400
author: Rúnar
comments: true
categories:
commentIssueId: 17
---

I've found that if I'm using `scala.concurrent.Future` in my code, I can get some really easy performance gains by just switching to `scalaz.concurrent.Task` instead, particularly if I'm chaining them with `map` or `flatMap` calls, or with `for` comprehensions.

## Jumping into thread pools

Every `Future` is basically some work that needs to be submitted to a thread pool. When you call `futureA.flatMap(a => futureB)`, both `Future[A]` and `Future[B]` need to be submitted to the thread pool, even though they are not running concurrently and could theoretically run on the same thread. This context switching takes a bit of time.

## Jumping on trampolines

With `scalaz.concurrent.Task` you have a bit more control over when you submit work to a thread pool and when you actually want to continue on the thread that is already executing a `Task`. When you say `taskA.flatMap(a => taskB)`, the `taskB` will by default just continue running on the same thread that was already executing `taskA`. If you explicitly want to dip into the thread pool, you have to say so with `taskB.fork`.

This works since a `Task` is not a concurrently running computation. It's a _description_ of a computation—a sequential list of instructions that may include instructions to submit some of the work to thread pools. The work is actually executed by a tight loop in `Task`'s `run` method. This loop is called a _trampoline_ since every step in the `Task` (that is, every subtask) returns control to this loop.

Jumping on a trampoline is a lot faster than jumping into a thread pool, so whenever we're composing `Future`s with `map` and `flatMap`, we can just switch to `Task` and make our code faster.

## Making fewer jumps

But sometimes we know that we want to continue on the same thread and we don't want to spend the time jumping on a trampoline at every step. To demonstrate this, I'll use the Ackermann function. This is not necessarily a good use case for `Future` but it shows the difference well.

```scala
// Only defined for positive `m` and `n`
def ackermann(m: Int, n: Int): Int = (m, n) match {
  case (0, _) => n + 1
  case (m, 0) => ackermann(m - 1, 1)
  case (m, n) => ackermann(m - 1, ackermann(m, n - 1))
}
```

This function is supposed to terminate for all positive `m` and `n`, but if they are modestly large, this recursive definition overflows the stack. We could use futures to alleviate this, jumping into a thread pool instead of making a stack frame at each step:

``` scala
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

def ackermannF(m: Int, n: Int): Future[Int] = {
  (m, n) match {
    case (0, _) => Future(n + 1)
    case (m, 0) => Future(ackermannF(m - 1, 1)).flatMap(identity)
    case (m, n) => for {
      x <- Future(ackermannF(m, n - 1))
      y <- x
      z <- Future(ackermannF(m - 1, y))
      r <- z
    } yield r
  }
}
```

Since there's no actual concurrency going on here, we can make this instantly faster by switching to `Task` instead, using a trampoline instead of a thread pool:

``` scala
import scalaz.concurrent.Task, Task._

def ackermannT(m: Int, n: Int): Task[Int] = {
  (m, n) match {
    case (0, _) => now(n + 1)
    case (m, 0) => suspend(ackermannT(m - 1, 1))
    case (m, n) => 
      suspend(ackermannT(m, n - 1)).flatMap { x =>
      suspend(ackermannT(m - 1, x)) }
  }
}
```

But even here, we're making too many jumps back to the trampoline with `suspend`. We don't actually need to suspend and return control to the trampoline at each step. We only need to do it enough times to avoid overflowing the stack. Let's say we know how large our stack can grow:

``` scala
val maxStack = 512
```

We can then keep track of how many recursive calls we've made, and jump on the trampoline only when we need to:

``` scala
def ackermannO(m: Int, n: Int): Task[Int] = {
  def step(m: Int, n: Int, stack: Int): Task[Int] =
    if (stack >= maxStack)
      suspend(ackermannO(m, n))
    else go(m, n, stack + 1)
  def go(m: Int, n: Int, stack: Int): Task[Int] =
    (m, n) match {
      case (0, _) => now(n + 1)
      case (m, 0) => step(m - 1, 1, stack)
      case (m, n) => for {
        internalRec <- step(m, n - 1, stack)
        result      <- step(m - 1, internalRec, stack)
      } yield result
    }
  go(m, n, 0)
}
```

## How fast is it?

I did some comparisons using [Caliper](https://github.com/google/caliper) and made this pretty graph for you:

{% img /images/futuretaskplot.png %}

The horizontal axis is the number of steps, and the vertical axis is the mean time that number of steps took over a few thousand runs.

This graph shows that `Task` is slightly faster than `Future` for submitting to thread pools (blue and yellow lines marked _Future_ and _Task_ respectively) only for very small tasks; up to about when you get to 50 steps, when (on my Macbook) both futures and tasks cross the 30 μs threshold. This difference is probably due to the fact that a `Future` is a running computation while a `Task` is partially constructed up front and explicitly `run` later. The overhead of the trampoline seems to catch up with whatever benefit that gives at around 50 steps. But honestly the difference between these two lines is not something I would care about in a real application.

But if we jump on the trampoline instead of submitting to a thread pool (green line marked _Trampoline_), things are _between one and two orders of magnitude faster_.

If we only jump on the trampoline when we really need it (red line marked _Optimized_), we can gain another order of magnitude.

If we measure without any trampolining at all, the line tracks the _Optimized_ red line pretty closely then shoots to infinity around 1000 (or however many frames fit in your stack space).

So if we're smart about trampolines vs thread pools, `Future` vs `Task`, and optimize for our stack size, we can go from milliseconds to microseconds with not very much effort. Or seconds to milliseconds, MHz to GHz, or weeks to hours, as the case may be.

