---
layout: post
title: "Objects, Identity, and Concept-formation"
date: 2008-12-04
comments: true
categories: 
author: Rúnar
commentIssueId: 10
---

Coming from a background in Pascal and C, during the 1990s, like most others, I became infatuated with Object-Oriented programming. I thought they were really on to something. It seemed intuitive. I read and re-read the [GoF book](http://www.amazon.com/Design-Patterns-Object-Oriented-Addison-Wesley-Professional/dp/0201633612). I became fluent in UML. I read Chris Alexander. I wrote factories and metafactories, and more class hierarchies than I'll ever count. I embraced Java eagerly, and XML moreso. But with experience came the realisation that I was solving the same problems over and over again. There was [some governing principle](http://en.wikipedia.org/wiki/Combinatory_logic) that I was missing but knew I had to grasp if I were to progress in my chosen profession. I went back to the fundamentals. I learned about LISP, lamdba calculi, type theory, relational programming, [category theory](http://en.wikipedia.org/wiki/Category_theory), and [epistemology](http://en.wikipedia.org/wiki/Introduction_to_Objectivist_Epistemology). And I'm still learning. But in integrating all of that into my existing knowledge, I discovered something that I found liberating:

_There is no such thing as Object-Oriented programming._

I realise that I might risk starting a religious war here, but I don't intend to. If you think I'm attacking you personally in some way, please stop reading and come back later with fresh eyes. Note that **I'm not saying that OO is a bad thing**. I'm saying that there isn't any special kind of programming that is object-oriented as against programming that isn't. It is a useless distinction, in exactly the same way that ["race"](http://freedomkeys.com/ar-racism.htm) is a useless distinction. And holding on to a useless concept isn't good for anybody. The term "object-oriented" is at least honest in that it says what it implies, which is a frame of mind, an orientation of the programmer. However, the practical effect of this orientation is anti-conceptual, in that it displaces actual concepts like algebra, calculus, closure, function, identity, type, value, variable, etc. It fosters an aversion to abstraction. So you could say that there are object-orientated programmers.

## "Object-Oriented" as a non-concept ##
Remember, you cannot be called upon to prove a negative. The burden of proof falls on whomever is claiming that there is such a thing as OO. But try as you might, [there's no objective definition of what "object-oriented" refers to](http://www.paulgraham.com/reesoo.html). It means anything you want, which is to say that it's rather meaningless. Peddlers of "object methodology" readily admit that "object-oriented" means different things to different people. To quote Aristotle: "Not to have one meaning is to have no meaning, and if words have no meaning, our reasoning with one another, and indeed with ourselves, has been annihilated."

When I say that there's no such thing as OO, I mean, more precisely, that there exists some abstraction (or several) that is referred to as "object-oriented", but that this abstraction has no actual referent in reality. It is an abstraction that is made in error. It is not necessary, and serves no cognitive purpose. It is "not even false".

This is to say that it's not like the concept of Santa Claus, which is a fiction, a falsehood. OO is not a fiction, but a misintegration. It is a term that sounds like a concept, but denotes nothing more than a vague collection of disparate elements, any of which can be omitted, and none of which are essential.

## A Proper Method of Concept-Formation ##

How is Object-Oriented a non-concept? Let's demonstrate that very cursorily. First, we have to explicitly identify what it means for a concept to be valid.

_Valid concepts are arrived at by induction. _Induction is the process of integrating facts of reality by observing similarities and omitting particulars. Or, if you will, the process of applying logic to experience. Our induction must adhere to the corollary laws of [identity](http://en.wikipedia.org/wiki/Law_of_identity), [noncontradiction](http://en.wikipedia.org/wiki/Law_of_noncontradiction), and [the excluded middle](http://en.wikipedia.org/wiki/Law_of_excluded_middle), and it must observe [causality](http://aynrandlexicon.com/lexicon/causality.html).

Note that a correct concept of concepts is the vehicle and not the cargo of this post. In brief, a valid concept or theory must meet the following criteria (hat tip: [David Harriman](http://www.theobjectivestandard.com/issues/2007-spring/induction-experimental-method.asp)):

1. It must be derived from observations by a valid method (by logical induction from experience or experiment as described above). A valid concept can have no arbitrary elements and no relations that are unsupported by the facts being integrated.
2. It must form an integrated whole in which every part supports every other. It cannot be a conglomeration of independent parts that are freely adjusted to fit a given situation.
3. It must be no more and no less general than required to meet criteria 1 and 2.

**OO doesn't meet criterion #1**. There's nothing about the nature of programming that gives rise to a concept of object-orientation. Rather, the notion is sparked by the desire to make an analogy between manipulating physical objects in the real world and manipulating abstract entities in the computer. The motivation is understandable, to help people think about programs in the same way that they are used to reasoning about physical objects. But the analogy isn't warranted by the facts, and therefore it is arbitrary. The concepts simply don't transfer, because they are abstractions over entities of a different kind. The mistake is to use the concepts anyway, choosing the illusion that software is like physical systems rather than understanding the actual nature of programs.

**OO doesn't meet criterion #2**: There's no consistent definition of object-orientation. It's a loose grab-bag in which nothing is essential. Even if you start removing things, it's not at all clear at what point your programming stops being object-oriented. One would think that it's at the point where you stop programming with objects, but that's merely begging the question since it's not clear what kind of entity is an object and what kind of entity isn't. We are to rely on intuition alone in discovering that. What's more, some of the contents of the "object-oriented" bag contradict one another (both inheritance and mutation contradict encapsulation, for example).

## Repairing the Disorientation ##

Of course, it does no good to tear down the cognitive package-deal that is "object-oriented" if you don't replace it with something. If you entered a village of [little blue creatures](http://www.smurf.com/) and convinced them that the term "smurf" had no cognitive utility, you would have to teach them a whole host of concepts to use in its stead. There are two main ideas that the anti-concept "object-oriented" annihilates:

1. It introduces the false dichotomy of identity as apart from equivalence.
2. It blurs the distinction between values and variables.

I will elaborate on each of these points in two follow-up posts. Stay tuned.

