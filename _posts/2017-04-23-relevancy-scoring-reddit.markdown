---
layout: post
title: "Relevancy scoring reddit to find related subs"
date:   2017-04-23 00:00:00
---

I spend so much free time on reddit. So much, that I decided what I really needed was a way to spend _even more time_ browsing it.

So, naturally, I decided to download some reddit data and try to find some new subs. In doing so, I encountered a common problem with these sorts of recommendation situations. Often, recommendations are for what everybody does, rather than being personalized for me and my situation. Relevancy scoring is a simple way to avoid this problem.

Code
----

[You can find the code on github, of course.](https://github.com/ririw/subreddit-scoring)

Demo: type a subreddit
----------------------
This takes a few seconds to load, it needs to pull in the subreddit data.

<iframe src='/assets/demo.html' style='width:100%; height:512px; border: 0'></iframe>

Relevancy scoring
-----------------
Let's start with a really simple approach to the problem of finding new subreddits:

> What other subreddits do people who comment on /r/programming most often comment on?

The answer is disappointing: they comment on the subs that _everyone_ comments on. The top 5 subreddits most commented on are:
 
 - AskReddit
 - funny
 - pics
 - worldnews
 - todayilearned

**Lame.**

Scoring with probability
------------------------

How about something a bit more refined?

> For those who comment on /r/programming, which subreddits do they comment on _unusually_ often?

This is called the relevancy score, and is simply `P(comments on sub|comments on programming) / P(comments on sub)`.

This doesn't quite work either. The top 5 are:

 - learnbasketball
 - Hahahahaha
 - MozillaTech
 - VeganFood
 - Clear_Void

Now, I've never heard of any of these subs. The problem here is that we're looking at cases with one or two comments, which means their scores are massive. The commonplace solution would be to add some "smoothing" to the denominator, making less-likely subs appear less often, but it's not quite right.

Thinking about probability
--------------------------
I find that in my work, many, many questions in machine learning or data analysis can be solved by thinking of the problem probabalistically. In this case, we asked for some measure of how more relevant a subreddit is to a set of users, compared to the general baseline. 

The mistake here is that we don't have accurate probabilities. For these smaller subreddits, we estimate the probability of a visit to them by looking at the number of visits to them, but even small amounts of noise in visitation will change the results massively.

Instead, we need to ask:

> What is the probaility of observing this level of commenting, by /r/programmer commenters, compared to baseline.

By re-phrasing as a probability, and including the possiblity of randomness in our measurements (comments), we're able to solve the problem nicely!

I calculate _how many standard deviations from the expected commenting level is the observed commenting level_. In code:

{% highlight python %}
    def score_sub(sub_id, given_sums, means):
        """
        sub_id: the ID of the subreddit of interest
        given_sums: the number of visits to different subs by people who visit /r/programming
        means: the global expected visit rate for different subs
        """
        # prepare a binomial distribution, assuming the "global" case
        dist = stats.binom(given_sums.sum(), means[sub_id])
        # Find its average and std
        mu = dist.mean()
        sd = dist.std()
        # and calculate how many std-devs away our observed level of visitation is.
        dist = (given_sums[sub_id] - mu) / sd

        return dist
{% endhighlight %}

This works perfectly. The top 5 results are:

 - linux
 - ProgrammerHumor
 - compsci
 - Python
 - cpp

Other examples:

 - politics: technology, worldnews, news, science, atheism
 - funny: pics, AdviceAnimals, gifs, todayilearned, WTF
 - books: television, explainlikeimfive, booksuggestions, news, literature



Conclusion
----------

This sort of relevancy scoring can be great for when you want simple recommendations, based on 
single items. They help work around the problem of popularity, where a recommendation tends to simply be what's popular with everyone, and zoom in on the things that are particular to a group. At the same time, they aren't overwhelmed by noise.

