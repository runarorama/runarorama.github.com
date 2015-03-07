---
layout: post
title: "A better reading list with Mathematica"
date: 2015-02-08 11:04
author: Rúnar
comments: true
categories:
commentIssueId: 13
---

Like a lot of people, I keep a list of books I want to read. And because there are a great many more books that interest me than I can possibly read in my lifetime, this list has become quite long.

In the olden days of brick-and-mortar bookstores and libraries, I would discover books to read by browsing shelves and picking up what looked interesting at the time. I might even find something that I knew was on my list. "Oh, I've been meaning to read that!"

The Internet changes this dynamic dramatically. It makes it much easier for me to discover books that interest me, and also to access any book that I might want to read, instantly, anywhere. At any given time, I have a couple of books that I'm "currently reading", and when I finish one I can start another immediately. I use Goodreads to manage my to-read list, and it's easy for me to scroll through the list and pick out my next book.

But again, this list is very long. So I wanted a good way to filter out books I will really never read, and sort it such that the most "important" books in some sense show up first. Then every time I need a new book I could take the first one from the list and make a binary decision: either "I will read this right now", or "I am never reading this". In the latter case, if a book interests me enough at a later time, I'm sure it will find its way back onto my list.

The problem then is to find a good metric by which to rank books. Goodreads lets users rank books with a star-rating from 1 to 5, and presents you with an average rating by which you can sort the list. The problem is that a lot of books that interest me have only one rating and it's 5 stars, giving the book an "average" of 5.0. So if I go with that method I will be perpetually reading obscure books that one other person has read and loved. This is not necessarily a bad thing, but I do want to branch out a bit.

Another possibility is to use the number of ratings to calculate a [confidence interval](http://en.wikipedia.org/wiki/Binomial_proportion_confidence_interval) for the average rating. For example, using the [Wilson score](http://www.evanmiller.org/how-not-to-sort-by-average-rating.html) I could find an upper and lower bound `s1` and `s2` (higher and lower than the average rating, respectively) that will let me say "I am 95% sure that any random sample of readers of an equal size would give an average rating between `s1` and `s2`." I could then sort the list by the lower bound `s1`.

But this method is dissatisfactory for a number of reasons. First, it's not clear how to fit star ratings to such a measure. If we do the naive thing and count a 1-star rating as 1/5 and a 5 star rating as 5/5, that counts a 1-star rating as a "partial success" in some sense. We could discard 1-stars as 0, and count 2, 3, 4, and 5 stars as 25%, 50%, 75%, and 100%, respectively.

But even if we did make it fit somehow, it turns out that if you take _any moderately popular book_ on Goodreads at random, it will have an average rating somewhere close to 4. I could [manufacture a prior](http://stephsun.com/silverizing.html) based on this knowledge and use that instead of the normal distribution or the Jeffreys prior in the confidence interval, but that would still not be a very good ranking because [reader review metascores are meaningless](http://stephsun.com/metascores.html).

In the article ["Reader review metascores are meaningless"](http://stephsun.com/metascores.html), Stephanie Shun suggests using the _percentage of 5-star ratings_ as the relevant metric rather than the average rating. This is a good suggestion, since even a single 5-star rating carries a lot of actionable information whereas an average rating close to 4.0 carries very little.

I can then use the Wilson score directly, counting a 5-star rating as a successful trial and any other rating as a failed one. I can then just use the normal distribution instead of working with an artisanally curated prior.

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

`Ratings` will give us an association list with the number of ratings for each score from 1 to 5, together with the total. For example, for the first book in their catalogue, _Harry Potter and the Half-Blood Prince_, here are the scores:

{% codeblock lang:mathematica %}
In[1]:= Ratings[1]

Out[1]= <|"5"     ->  782740,
          "4"     ->  352725,
          "3"     ->  111068,
          "2"     ->   17693,
          "1"     ->    5289,
          "total" -> 1269515|>
{% endcodeblock %}

Sweet. Let's see how Harry Potter #6 would score with our rating:

{% codeblock lang:mathematica %}
In[2]:= Wilson[Out[1]["5"], Out[1]["total"], 0.95]

Out[2]= 0.61572
{% endcodeblock %}

So Wilson is 95% confident that in any random sample of about 1.2 million Harry Potter readers, at least 61.572% of them would give _The Half-Blood Prince_ a 5-star rating. That turns out to be a pretty high score, so if this book were on my list (which it isn't), it would feature pretty close to the very top.

But now the score for a relatively obscure title is too low. For example, the lower bound of the 95% confidence interval for a single-rating 5-star book will be 0.206549, which will be towards the bottom of any list. This means I would never get to any of the obscure books on my reading list, since they would be edged out by moderately popular books with an average rating close to 4.0.

See, if I've picked a book that _I want to read_, I'd consider five ratings that are all five stars a much stronger signal than the fact that people who like Harry Potter enough to read 5 previous books loved the 6th one. Currently the 5*5 book will score 57%, a bit weaker than the Potter book's 62%.

I can fix this by lowering the confidence level. Because honestly, I don't need a high confidence in the ranking. I'd rather err on the side of picking up a deservedly obscure book than to miss out on a rare gem. Experimenting with this a bit, I find that a confidence around 80% raises the obscure books enough to give me an interesting mix. For example, a 5*5 book gets a 75% rank, while the Harry Potter one stays at 62%.

I'm going to call that the _Rúnar rank_ of a given book. The Rúnar rank is defined as the lower bound of the _1-1/q_ Wilson confidence interval for scoring in the *q*th *q*-quantile. In the special case of Goodreads ratings, it's the 80% confidence for a 5-star rating.

{% codeblock lang:mathematica %}
RunarRank[id_] := With[{ratings = Ratings[id]},
  Wilson[ratings["5"], ratings["total"], .8]]]
{% endcodeblock %}

Unfortunately, there's no way to get the rank of all the books in my reading list in one call to the Goodreads API. And when I asked them about it they basically said "you can't do that", so I'm assuming that feature will not be added any time soon. So I'll have to get the reading list first, then call `RunarRank` for each book's `id`. In Goodreads, books are managed by "shelves", and the API allows getting the contents of a given shelf, 200 books at a time:

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

I'm doing a bunch of XML pattern matching here to get the `id`, `title`, `average_rating`, and first `author` of each book. Then I put that in an association list. I'm getting only the top-200 books on the list by average rating (which currently is about half my list).

With that in hand, I can get the contents of my "to-read" shelf with `GetShelf[runar, "to-read"]`, where `runar` is my Goodreads user id. And given that, I can call `RunarRank` on each book on the shelf, then sort the result by that rank:

{% codeblock lang:mathematica %}
RunarSort[shelf_] :=
 Sort[Map[Function[x, Append[x, "rank" -> RunarRank[x["id"]]]], 
   shelf], #1["rank"] > #2["rank"] &]
{% endcodeblock %}

To get the ranked reading list of any user:

{% codeblock lang:mathematica %}
ReadingList[user_] := RunarSort[GetShelf[user, "to-read"]]
{% endcodeblock %}

And to print it out nicely:

{% codeblock lang:mathematica %}
Gridify[books_] := 
 Grid[Flatten[
   Cases[books, 
    b_ -> {
      {b["id"], b["title"], UnitConvert[b["rank"], "Percent"]},
      {"", b["author"], b["avg"]}, {"", "", ""} }], 1],
   Alignment -> Left]
{% endcodeblock %}

Now I can get, say, the first 10 books on my improved reading list:

{% codeblock lang:mathematica %}
Gridify[ReadingList[runar][[1 ;; 10]]]
{% endcodeblock %}

| | | |
|:---|:---|---:|
| 9934419 | Kvæðasafn | 75.2743% | 
|  | Snorri Hjartarson | 5.00 | 
|&nbsp;|
| 17278 | The Feynman Lectures on Physics Vol 1 | 67.2231% | 
|  | Richard P. Feynman | 4.58 | 
|&nbsp;|
| 640909 | The Knowing Animal: A Philosophical Inquiry Into Knowledge and Truth | 64.6221% | 
|  | Raymond Tallis | 5.00 | 
|&nbsp;|
| 640913 | The Hand: A Philosophical Inquiry Into Human Being | 64.6221% | 
|  | Raymond Tallis | 5.00 | 
|&nbsp;|
| 4050770 | Volition As Cognitive Self Regulation | 62.231% | 
|  | Harry Binswanger | 4.86 | 
|&nbsp;|
| 8664353 | Unbroken: A World War II Story of Survival, Resilience, and Redemption | 60.9849% | 
|  | Laura Hillenbrand | 4.45 | 
|&nbsp;|
| 13413455 | Software Foundations | 60.1596% | 
|  | Benjamin C. Pierce | 4.80 | 
|&nbsp;|
| 77523 | Harry Potter and the Sorcerer's Stone (Harry Potter #1) | 59.1459% | 
|  | J.K. Rowling | 4.39 | 
|&nbsp;|
| 13539024 | Free Market Revolution: How Ayn Rand's Ideas Can End Big Government | 59.1102% | 
|  | Yaron Brook | 4.48 | 
|&nbsp;|
| 1609224 | The Law | 58.767% | 
|  | Frédéric Bastiat | 4.40 | 
|&nbsp;|

I'm quite happy with that. Some very popular and well-loved books interspersed with obscure ones with exclusively (or almost exclusively) positive reviews. The most satisfying thing is that the rating carries a real meaning. It's basically the relative likelihood that I will enjoy the book enough to rate it five stars.

I can test this ranking against books I've already read. Here's the top of my "read" shelf, according to their Rúnar Rank:

|    |    |    |
|:---|:---|---:|
|17930467|The Fourth Phase of Water|68.0406%|
||Gerald H. Pollack|4.85|
|&nbsp;|
|7687279|Nothing Less Than Victory: Decisive Wars and the Lessons of History|64.9297%|
||John David Lewis|4.67|
|&nbsp;|
|43713|Structure and Interpretation of Computer Programs|62.0211%|
||Harold Abelson|4.47|
|&nbsp;|
|7543507|Capitalism Unbound: The Incontestable Moral Case for Individual Rights|57.6085%|
||Andrew Bernstein|4.67|
|&nbsp;|
|13542387|The DIM Hypothesis: Why the Lights of the West Are Going Out|55.3296%|
||Leonard Peikoff|4.37|
|&nbsp;|
|5932|Twenty Love Poems and a Song of Despair|54.7205%|
||Pablo Neruda|4.36|
|&nbsp;|
|18007564|The Martian|53.9136%|
||Andy Weir|4.36|
|&nbsp;|
|24113|Gödel, Escher, Bach: An Eternal Golden Braid|53.5588%|
||Douglas R. Hofstadter|4.29|
|&nbsp;|
|19312|The Brothers Lionheart|53.0952%|
||Astrid Lindgren|4.33|
|&nbsp;|
|13541678|Functional Programming in Scala|52.6902%|
||Rúnar Bjarnason|4.54|
|&nbsp;|

That's perfect. Those are definitely books I thouroughly enjoyed and would heartily recommend. Especially that last one.


