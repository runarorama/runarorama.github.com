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

I can then use the Wilson score directly, counting a 5-star rating as a successful trial and any other rating as a failed one. We can then just use the normal distribution instead of working with an artisanally curated prior.

Mathematica makes it easy to generate the Wilson score. Here, `pos` is the number of positive trials (number of 5-star ratings), `n` is the number of total ratings, and `confidence` is the desired confidence percentage. I'm taking the lower bound of the confidence interval to get my score.

{% codeblock lang:mathematica %}
Wilson[pos_, n_, confidence_] := 
  If[n == 0 || n < pos, 0, 
    With[{
      z = Quantile[NormalDistribution[], 1 - (1 - confidence)/2], 
      p = pos/n
    },
    (-(z Sqrt[(z^2/(4 n) + (1 - p) p)/n]) + z^2/(2 n) + p) /
      (z^2/n + 1)
    ]
  ]
{% endcodeblock %}

Now I just need to get the book data from Goodreads. Fortunately, [it has a pretty rich API](https://www.goodreads.com/api). I just need a developer key, which anyone can get for free.

For example, to get the ratings for a given book `id`, we can use their XML api for books and pattern match on the result to get the ratings by score:

{% codeblock lang:mathematica %}
Ratings[id_] := Ratings[id] = 
  With[{
    raw = Cases[Import[ToString[StringForm[
        "http://www.goodreads.com/book/show.xml?key=``&id=``", key, id]]],
      XMLElement["rating_dist", _, {d_}] -> d, Infinity]
  }, 
  With[{
    data = StringSplit[#, ":"] & /@ StringSplit[raw, "|"]
  }, Block[{}, Pause[1];
  FromDigits /@ Association[Rule @@@ First[data]]]]];
{% endcodeblock %}

Here, `key` is my Goodreads developer API key, defined elsewhere. I put a `Pause[1]` in the call since Goodreads throttles API calls so you can't make more than one call per second to each API endpoint. I'm also memoizing the result, by assigning to `Ratings[id]` in the global environment.

`Ratings` will give us an association list with the number of ratings for each score from 1 to 5, together with the total. For example, for the first book in their catalogue, "Harry Potter and the Half-Blood Prince", here are the scores:

{% codeblock lang:mathematica %}
In[1]:= Ratings[1]

Out[1]= <|"5" -> 782740, "4" -> 352725, "3" -> 111068,
         "2" -> 17693, "1" -> 5289, "total" -> 1269515|>
{% endcodeblock %}

Sweet. Let's see how Harry Potter #6 would score with our rating:

{% codeblock lang:mathematica %}
In[2]:= Wilson[Out[1]["5"], Out[1]["total"], 0.95]

Out[2]= 0.61572
{% endcodeblock %}

So Wilson is 95% confident that in any random sample of about 1.2 million users, 62% of them would give this book a 5-star rating. That turns out to be a pretty high score, so if this book were on my list (which it isn't), it would feature pretty close to the very top.

But now the lower bound for a relatively obscure title is pretty low. See, if I've picked a book that _I want to read_, I'd consider a single five-star rating to not carry much signal, but if it has five ratings that are all five stars, I'd consider that a strong signal, moreso than the fact that Harry Potter has a lot of fans. So I'll introduce a scaling factor which will have the effect that 5-star ratings on a book with few total ratings weigh more than those on a book with a lot of ratings:

{% codeblock lang:mathematica %}
ScaledWilson[pos_, n_, confidence_, scaling_] := 
   Wilson[pos*scaling, n*scaling, confidence]
{% endcodeblock %}

Experimenting with this a bit, I find that a scaling factor of 2 or so gives the appropriate amount of weight to more obscure books. But this is entirely subjective.

We can now get the "true rank" of any book:

{% codeblock lang:mathematica %}
TrueRank[id_] := With[{ratings = Ratings[id]},
  ScaledWilson[ratings["5"], ratings["total"], 0.95, 2]]]
{% endcodeblock %}

Unfortunately, there's no way to get the ratings of all the books in my reading list in one fell swoop. I'll have to get the reading list first, then call `Rating` for each book's `id`. In Goodreads, books are managed by "shelves", and the API allows getting the entire contents of a given shelf:

{% codeblock lang:mathematica %}
GetShelf[user_, shelf_] :=
  With[{raw = 
    Cases[Import[
      ToString[
       StringForm[
        "http://www.goodreads.com/review/list.xml?v=2&key=``&id=``&\
shelf=``&sort=avg_rating&order=d&per_page=200", key, user, shelf]]], 
     XMLElement[
       "book", _, {___, XMLElement["id", ___, {id_}], ___, 
        XMLElement["title", _, {title_}], ___, 
        XMLElement["average_rating", _, {avg_}], ___, 
        XMLElement[
         "authors", _, {XMLElement[
           "author", _, {___, 
            XMLElement[
             "name", _, {author_}], ___}], ___}], ___}] -> {"id" -> 
        id, "title" -> title, "author" -> author, "avg" -> avg}, 
     Infinity]}, Association /@ raw]
{% endcodeblock %}

I'm doing a bunch of XML pattern matching here to get the `id`, `title`, `average_rating`, and first `author` of each book. Then I put that in an association list.

With that in hand, I can get the contents of my "to-read" shelf with `GetShelf[runar, "to-read"]`, where `runar` is my Goodreads user id. And given that, I can call `TrueRank` on each book on the shelf, then sort the result by that rank:

{% codeblock lang:mathematica %}
TrueSort[shelf_] :=
 Sort[Map[Function[x, Append[x, "rank" -> TrueRank[x["id"]]]], 
   shelf], #1["rank"] > #2["rank"] &]
{% endcodeblock %}

To get the ranked reading list of any user:

{% codeblock lang:mathematica %}
ReadingList[user_] := TrueSort[GetShelf[user, "to-read"]]
{% endcodeblock %}

And to print it out nicely:

{% codeblock lang:mathematica %}
Gridify[books_] := 
 Grid[Cases[books, 
   b_ -> {b["id"], b["title"], b["author"], b["rank"], b["avg"]}], 
  Alignment -> Left]
{% endcodeblock %}

Now I can get, say, the first 10 books on my improved reading list:

{% codeblock lang:mathematica %}
Gridify[ReadingList[runar][[1 ;; 10]]]
{% endcodeblock %}

{% codeblock %}
5546     The Feynman Lectures on Physics (0.678576, 4.58)
         Richard P. Feynman

16157938 Concepts and Their Role in Knowledge: Reflections on Objectivist Epistemology (0.675592, 5.00)
         Allan Gotthelf

8664353  Unbroken: A World War II Story of Survival, Resilience, and Redemption (0.609754,4.44)
         Laura Hillenbrand

640909   The Knowing Animal: A Philosophical Inquiry Into Knowledge and Truth (0.609666, 5.00)
         Raymond Tallis

640913   The Hand: A Philosophical Inquiry Into Human Being (0.609666, 5.00)
         Raymond Tallis

4050770  Volition As Cognitive Self Regulation (0.600586, 4.86)
         Harry Binswanger 

13539024 Free Market Revolution: How Ayn Rand's Ideas Can End Big Government (0.587592, 4.48)
         Yaron Brook 

1609224  The Law (0.586947, 4.40)
         Frédéric Bastiat

304079   The Essential Rumi (0.580289, 4.42)
         Rumi

151848   Probability Theory: The Logic of Science (0.577267, 4.48)
         E.T. Jaynes
{% endcodeblock %}

I'm quite happy with that. Some very popular and well-loved books interspersed with obscure ones with exclusively (or almost exclusively) positive reviews. I'll get cracking on Feynman, then.

