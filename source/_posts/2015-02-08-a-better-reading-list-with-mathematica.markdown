---
layout: post
title: "A better reading list with Mathematica"
date: 2015-02-08 11:04
author: Rúnar
comments: true
categories:
commentIssueId:
---

Like a lot of people, I keep a list of books I want to read. And because there are a great many more books that interest me than I can possibly read in my lifetime, this list has become quite long.

In the olden days of brick-and-mortar bookstores and libraries, I would discover books to read by browsing shelves and picking up what looked interesting at the time. I might even find something that I knew was on my list. "Oh, I've been meaning to read that!"

The Internet changes this dynamic dramatically. It makes it much easier for me to discover books that interest me, and also to access any book that I might want to read, instantly, anywhere. At any given time, I have a couple of books that I'm "currently reading", and when I finish one I can start another immediately. I use Goodreads to manage my to-read list, and it's easy for me to scroll through the list and pick out my next book.

But again, this list is very long. So I wanted a good way to filter out books I will really never read, and sort it such that the most "important" books in some sense show up first. Then every time I need a new book I could take the first one from the list and make a binary decision: either "I will read this right now", or "I am never reading this". In the latter case, if a book interests me enough at a later time, I'm sure it will find its way back onto my list.

The problem then is to find a good metric by which to rank books. Goodreads lets users rank books with a star-rating from 1 to 5, and presents you with an average rating by which you can sort the list. The problem is that a lot of books that interest me have only one rating and it's 5 stars, giving the book an "average" of 5.0. So if I go with that method I will be perpetually reading obscure books that one other person has read and loved. This is not necessarily a bad thing, but I do want to branch out a bit.

Another possibility is to use the number of ratings to calculate a [confidence interval](http://en.wikipedia.org/wiki/Binomial_proportion_confidence_interval) for the average rating. For example, using the [Wilson score](http://www.evanmiller.org/how-not-to-sort-by-average-rating.html) I could find an upper and lower bound `s1` and `s2` (higher and lower than the average rating, respectively) that will let me say "I am 95% sure that any random sample of readers of an equal size would give an average rating between `s1` and `s2`." I could then sort the list by the lower bound `s1`.

But this method is dissatisfactory for a number of reasons. Firstly, the lower bound of the confidence interval for a book with only one 5-star rating will be 2.82759. This means I would never get to any of the obscure books on my list, since they would be edged out by books with a solid and confident rating of 3 out of 5.

What's more, it turns out that if you take _any moderately popular book_ on Goodreads at random, it will have an average rating somewhere close to 4. I could [manufacture a prior](http://stephsun.com/silverizing.html) based on this knowledge and use that instead of the normal distribution in the confidence interval, but that would still not be a very good ranking because [reader review metascores are meaningless](http://stephsun.com/metascores.html).

In the article ["Reader review metascores are meaningless"](http://stephsun.com/metascores.html), Stephanie Shun suggests using the _percentage of 5-star ratings_ as the relevant metric rather than the _average rating_. This is a good suggestion, since even a single 5-star rating carries a lot of actionable information whereas an average rating close to 4.0 carries very little.

I can then use the Wilson score directly, counting a 5-star rating as a successful trial and any other rating as a failed one.

{% codeblock lang:mathematica %}
Wilson[pos_, n_, confidence_] := 
 If[n == 0 || n < pos, 0, 
  With[{z = Quantile[NormalDistribution[], 1 - (1 - confidence)/2], 
    p = pos/n}, (-(z Sqrt[(z^2/(4 n) + (1 - p) p)/n]) + z^2/(2 n) + 
      p)/(z^2/n + 1)]]
{% endcodeblock %}

