---
layout: post
title: "Recommending movies with deep learning"
date: Sun Sep 25 20:26:07 AEST 2016
---
_Check out the accompanying [notebook]({{site.url}}/assets/Recommending movies.ipynb)_

One of the things I like about deep neural networks and all the accompanying tools, are their flexibility. There's so much more to them than just classification! All it takes is a set of mental shortcuts, that change the way you look at a neural network, and suddenly you start to see many new ways to approach deep learning problems.

The shortcut I'll talk about in this post is _embeddings_. Embeddings let you take discrete items (like movies or books or words or whatever), and transform them into vectors. These vectors are generally more meaningful than the items themselves. We'll be creating embeddings for movies and users, and using them to predict the rating a user would have given to a particular movie.

I'll also try to include some of the things that _didn't_ work, and some of the things I did to debug the system. Those tricks are perhaps more useful than the final technique, and they don't often come up in finished papers.

# The data
For these experiments I'm using the [movielens 1M dataset](http://grouplens.org/datasets/movielens/). The dataset is basically people's ratings of movies they've watched, from one to five stars. It also includes a few other bits of information that I'm not using (yet), such as genre of movie or viewer profession.

# Alternative approaches

As far as I know, the most common way to approach these sorts of problems are as matrix decomposition factorization problems. In matrix decomposition problems, you start with the fat user-movie matrix. Every user has a cell for every movie, and where they've rated a film, that cell will have their rating, otherwise it'll be zero.

You take this fat user-movie matrix, and you decompose it into two skinny matrices: one for users and one for movies. The user one will have an entry for every user, and the movie one will have an entry for every movie. Each entry, in either matrix, will have a certain, pre-specified length, called the inner dimension. There will most likely be some conditions on the two matrices. In Nonnegative matrix factorization, for example, you optimize the squared error between the inner product of the skinny matrices, and the fat matrix. You do so, while requiring that the two decomposed matrices may only have positive or zero entries.

# My approach

The approach I took blends deep learning and matrix decomposition methods. To my mind, the disadvantage of matrix decompositions are:

 - They often aren't flexible in how they re-combine the decomposed matrices
 - They don't allow for the addition of extra information (more on this in another post).

Instead of doing a matrix decomposition, I'll be learning an embedding for both users and movies, as well as a deep-net that combines them into an answer. It's still a classification problem (classifying how many stars a movie got), but there's a whole lot of interesting stuff in the embedding.

## Thinking about deep learning

My favorite thing about deep learning is that it's flexibility means there are so may ways to think about a problem. The three mental frameworks I generally use are:

1. Structure a reward to make a good model. Generally this is for unsupervised settings, or cases where I don't have good data. In these cases, I spend a lot of time thinking about how to set up a problem so that it can easily be learned. For examples, take a look at contrastive divergence and RBMs, generative adversarial networks, DeepFace, and word2vec. All super-neat examples of where setting up a problem in a clever, not necessarily obvious way can reap dividends.
2. Network structures should match problem structures. Often, the way a network is put together should, in some way, match the problem. I'll generally look for ways to dive the problem up, or existing network structures to try to improve performance. This would be the main technique I'm using in this blog post. For some examples, check out the papers on machine translation, where clever training and network structure is used to make translators for different languages.
3. Learn more statistics. This is a framework that I wish I could use more, because I can see it's a powerful tool. For examples, check out [this post](http://mlg.eng.cam.ac.uk/yarin/blog_3d801aa532c1ce.html), or [this blog](http://www.inference.vc/)

Anyway, this post deals entirely with frameworks one and two. The reward structure isn't too hard, there are a couple of options:

 - Output a single number, and optimize mean squared error between the output and the rating given. For this one, I'd also suggest a special "rounding" version. We know we're limited to 1, 2, 3, 4 or 5 as an output. By rounding our result to the closest value, we can get a bit more performance for free.
 - Do a softmax classifier with five outputs.

In the end I tried both, and the second worked a little bit better. 

The network structure is a little more interesting. I used keras' embedding layer to produce vectors for both movies and users, and concatenated them together and fed them into a standard neural network.

# Things that went wrong

In no particular order:

- Regression with neural networks can be tricky. Probably the best tip I could give here would be to make a simpler version of the problem, and make sure you're able to make it work. If you can't do that it may mean that your network's output layers aren't right.
- I tried a bunch of tricks to try to speed up convergence with pre-training. For example, training the network only, learning the embedding vectors alone, so the network could "center" itself, and match its output distribution to the problem. None of these tricks worked.
- I wasted time fiddling with optimizers. Adam works really well, and leave these sorts of optimizations until the end. 
- I wasted heaps of time playing with layer sizes. This is something that comes towards the end of the whole development process (although it does make sense to play around with it a bit. You may just need a very big network).
- I didn't build test datasets right off the bat. Test datasets are immensely helpful. In the end, I made two skinny matrices, multiplied them together and then did some arithmetic to produce a fake "rating" to regress. This helped me learn a bunch about the network I was trying to build.

So my number one tip here: build a test version of your problem that is _so simple_ that your network will be able to solve it. Trust me, it'll help.

# Final structure
In the end, I used the following code to build my network:

{% highlight python %}
import keras.models as kmodels
import keras.layers as klayers
import keras.backend as K
import keras

movie_input = keras.layers.Input(shape=[1])
movie_vec = keras.layers.Flatten()(
	keras.layers.Embedding(n_movies + 1, 32)(movie_input))
movie_vec = keras.layers.Dropout(0.5)(movie_vec)

user_input = keras.layers.Input(shape=[1])
user_vec = keras.layers.Flatten()(
	keras.layers.Embedding(n_users + 1, 32)(user_input))
user_vec = keras.layers.Dropout(0.5)(user_vec)

input_vecs = keras.layers.merge([movie_vec, user_vec], mode='concat')
nn = keras.layers.Dropout(0.5)(
	keras.layers.Dense(128, activation='relu')(input_vecs))
nn = keras.layers.normalization.BatchNormalization()(nn)
nn = keras.layers.Dropout(0.5)(
	keras.layers.Dense(128, activation='relu')(nn))
nn = keras.layers.normalization.BatchNormalization()(nn)
nn = keras.layers.Dense(128, activation='relu')(nn)

result = keras.layers.Dense(5, activation='softmax')(nn)

model = kmodels.Model([movie_input, user_input], result)
model.compile('adam', 'categorical_crossentropy')
{% endhighlight %}

As you can see, I take `movie_input` and `user_input`. Both are single numbers, representing the user and movie ID. These are both turned into vectors with the keras `Embedding` layer. Both vectors have 32 entries.

Next, I smush them both together with a merge (notice that it's `merge` with a small m). Then I put them through a simple dense neural network, with dropout and batch normalization. Finally I take the result with softmax, and compile it all.

*You also check out the [full notebook]({{site.url}}/assets/Recommending movies.ipynb)*

# Results
It does pretty well! I used mean absolute error as my metric, so I could easily compare it to [these stats](http://www.mymedialite.net/examples/datasets.html). Their best result is 0.668 MAE, and after ten rounds of training and a 25% holdout set, I managed to reach 0.669, a little bit above their best effort.

Of course, MyMediaLite has one critical advantage: speed. Their system is built to allow for very fast lookups, whereas my one needs to run a complex neural network for every single movie. That'd be fine for _this_ application - there aren't too many movies, but take youtube for example: they [recently explained](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45530.pdf) their recommender, and mention having millions of videos and a billion users. Too big for a simple tool like mine. 

# Issues and improvements

- I'd like to include information about the movie and viewer to augment the embedded vector
- I'd like to find ways to balance memorization and semantic embedding. This was probably the biggest problem that the system has. It's very easy for it to do really well by "memorizing", and producing movie/user vectors that code for each other, rather than generalizing to "semantic" vectors about content. I'd love to find ways to do this better. Hit me up at richardweiss@richardweiss.org if you have ideas.
- It would be cool to include a summary of the movie, like its IMDB snippet and have the network read it and then use that information in recommendation.
- I'd also be quite interested in ways to regularize the embedding. The way I see it, the embedding will end up being a manifold (like a surface), with movies positioned over it. There may be ways to regularize these manifolds, and it would be interesting to see what effect that would have on performance.

# Summary

I learned a lot from building this, hopefully you did as well. The main takeaways are:

 - Always make test data.
 - Don't dwell too long on fiddling with the network structure, at least not in small ways. Try for the big wins first. In this case they were using dropout to prevent overfitting, and batch normalization to speed convergence
 - It's possible to use embedding for recommendation.
 - But there's a fine, fine line between embedding and memorization: a lot of the fiddling that I did was with dropout and layer sizes to discourage overfitting.

I'll experiment a little more, and try to re-visit some of the issues and improvements, to see what improvements I can  make, and what more I can learn from the problem.

Thanks for reading!
