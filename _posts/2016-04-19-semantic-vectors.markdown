---
layout: post
title: "Semantic vectorization"
date: Tue Apr 19 23:02:53 AEST 2016
---

I've been interested in chatbots, for work reasons. Something that really bugs me about
chatbots is that they can understand a scentence, but then will fail to understand a 
semantically identical scentence. For example 

> "What's the weather"

> "What is the weather like"

And one might be correctly interpretend, and the other might just confuse the bot. Note that
there is a subtle difference between difference and response, which is easy to confuse when
you talk about chatbots. Saying

> Is it raining

and

> I'd like to know if it's raining

elicit the same response, but the first is a direct statement, along the lines of "robot,
tell me if it is raining", while the second is far more implcit "robot, I'd like to know 
if it's raining, and my (implicit) hope is that you'll tell me the answer". 

Solutions
---------

What I'd like to do is produce a vectorization for small scentences like these that captures
their esscence, and ideally would let me generate similar scentences. A few options I'm 
considering

1. Produce logical formulae and then produce scentences from them, and try to learn a 
   projection into a vector space that preserves similarity between similar scentences.
   I don't like this one because it's very hard to do the autogeneration in a way that 
   really captures the complexity of language. Inevitably I'll just learn a small subset
   of the options.
2. Like words, but with scentences: word2vec did really well at this, and it was based 
   on predicting a word from its cohort. Perhaps we can do something similar. The problem
   with this is that scentences are so much more sparse than words, so it's hard to learn.
3. An autoencoder that produces scentences that match a particular input scentence. It is
   "scored" on how well the two scentences match each other, but regularized on how similar
   they are. This is like an adversarial generative model, but with a special regularizational
   term. The problem here is that adversarial nets still need a training set, although not
   as large a one as most deep learning problems.
