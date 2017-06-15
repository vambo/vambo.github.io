---
layout: post
title: The travelling tournament problem
---

A while ago I had to create a tournament schedule with the following constraints:

- Each team would play each of the other teams twice, once at home and once away
- The home and away games should alternate (if possible)

The first thing to come to mind is the [Round robin tournament system](https://en.wikipedia.org/wiki/Round-robin_tournament). However, while straightforward to implement, given 6 teams it appears one of the teams will have 4 home games out of 5 which is unacceptable. I knew that there's a better way. Not due to intuition, but because there are pre-made schedules available, such as [this one](http://sgvda.rhadt.com/data/forms/6&8&10sched.pdf).

Still, there's something unsatisfactory about using hard-coded tables and thus being limited to a few specific tournament sizes.

Some googling turns up [papers](http://repository.cmu.edu/cgi/viewcontent.cgi?article=1512&context=tepper) on the topic which go into graph theory and mention canonical 1-factorization as a solution. However, if one is not a mathematician and time is of the essence, transforming the formulas to an algorithm can prove difficult.

Thankfully, there's a book called [A Guide to Graph Colouring: Algorithms and Applications](http://www.springer.com/gp/book/9783319257280) which includes the following pseudocode:

<img src="/images/posts/2017-6-15/canonical-schedule.png">

The downside is that it's not entirely correct (which I notified the author of). 

Namely, line 9 should read:

```
if j is odd then..
```

I found that out again not due to great mathematical intuition, but by comparing the results of my implementation against the ones in the book, identifying a pattern in the differences, and tracing that back to a certain part of the algorithm.

[Here](https://bitbucket.org/snippets/vambo/XLKb7/roundrobin) is a Java implementation of the corrected pseudocode.

After having wrestled with this problem for a while I have new-found respect for the field of tournament scheduling, especially considering the additional requirements in the more prominent leagues, such as:
- distance (optimal travel distances for teams)
- time (venues being unavailable on certain dates, holidays, and competing events)
- different amounts of games against teams in and out of conference

Add the fact that unforeseen factors like airline strikes, terror attacks and natural disasters can require changes to the schedule, and suddenly managing all this is a full-time job (or probably several).






