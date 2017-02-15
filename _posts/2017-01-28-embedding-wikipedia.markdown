---
layout: post
title: "Embedding wikipedia with gensim"
date:   2017-01-28 10:10:00
---

# Background

One of the things that interests me the most about deep learning is how easy it is able to turn up patterns in unusual places.
It's always interesting to see an unexpected way of using deep learning methods for something other than the usual classification tasks.
Not that classification isn't interesting, but there's far more unlabeled data out there than there is labeled, and it's always good to see it being put to use.

In this case, I'd like to see if the wikipedia link graph can be used to infer a sort of "concept vector", a way to lay out concepts in space, much as you'd lay out a library.

# Theory
What I'd like to do is use `word2vec` to vectorize random walks over the wikipedia link graph. 
This will _hopefully_ yield vectors that relate the content of pages to each other. 
Using `word2vec` also means I don't have to fuss too much by writing my own tool.

You can access the code [on github](https://github.com/ririw/ririw.github.io/blob/master/assets/deep-networks/Deep%20graph%20embedding.ipynb).

# The data

For this task, I used a dbpedia dump of wikipedia from the Netherlands.
Why that one? Because it's the smallest wiki-dataset that I could find :).
The English one is just too big. 

Specifically, I use the one that's canonicalized to the _english_ page titles, so that when I want to understand what's going on I don't need to understand it in Danish.
You can download it from [dbpedia.org](http://wiki.dbpedia.org/downloads-2016-04).
Look for the 'ttl' format, `page_links_en_uris_nl.ttl.bz2`. 

Each line looks like this:

```<http://dbpedia.org/resource/Algorithm> <http://dbpedia.org/ontology/wikiPageWikiLink> <http://dbpedia.org/resource/Iteration> .```

My general plan is:

1. Walk through each line (ignoring comments, starting with '#'
2. Split by the space character
3. Grab the two URLs (not the middle one, it's a page-link type, and there's only one type)
4. Make a graph of undirected connections between the pages.
5. Sample walks from the graphs, of length 10.

This forms the basis for my dataset.

# Modeling

For the model, I simply used gensim's word2vec implementation, with its default arguments.
I also streamed the dataset, because it's very easy to generate random walks over the dataset.
Also note that I used _undirected_ walks. I found directed ones gave worse results.

# Results
The results are quite plausible! For example, I can ask about `Stone Cold Steve Austin`, and I get wrestling related results, like `Total_Nonstop_Action_Wrestling`,
 and `Hulk_Hogan`. Or I can search for flower:

{% highlight python %}
model.most_similar(u'Flower')
[(u'Leaf', 0.9957747459411621),
 (u'Petal', 0.992283821105957),
 (u'Lupinus_luteus', 0.9921330809593201),
 (u'Plant_stem', 0.9916960000991821),
 (u'Eudicots', 0.9907293915748596),
 (u'Category:Palearctic_flora', 0.989030122756958),
 (u'Prunella_vulgaris', 0.9888185858726501),
 (u'Trichome', 0.9884505271911621),
 (u'Leaf_shape', 0.9883744716644287),
 (u'Diplotaxis_muralis', 0.9879242181777954)]
{% endhighlight %}

I tried the positive/negative example, the one that made word2vec famous (`king + man - woman = queen`), but it didn't really work.
For starters, the dataset didn't have the word for King, but I tried `queen - woman + man` and got nothing of interest.

On the other hand, I did try `canada - north + south`, and `new zealand` was quite high up the list.
And New Zealand is the canada of the south, in many ways. 
I say this as an Australian.

# Conculsion
This worked pretty well! I think its value isn't in embedding _wikipedia_, but in embedding other graph-datasets, and embedding them onto a meaningful manifold. In particular, the "similar" query seems to be extremely promising.
