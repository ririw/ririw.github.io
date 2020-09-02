---
layout: post
title: "Deep learning applications on a raspberry pi using Onnx"
date:   2020-08-31 00:00:00
---

Summary
-------

I've recently made a tool to turn my smart lights on and off with an automated people counter, based on the [AMG8833](https://www.adafruit.com/product/3538), 
and a raspberry pi zero. For details on how I made the pi talk to the AMG8833, see the [previous blog post](/2020/08/31/rust-amg8833-on-pizero.html).

In this post, I'll outline:

1. the general approach I've used to build the people counter, using *pytorch* and *python*
2. how I used [onnx](http://onnx.ai/) to embed my *pytorch* model into a rust executable.

By the end of the process, I have a single executable that I can run on my pi zero. It gets good FPS - each frame takes about 30-40ms to process, plenty fast enough for me.

And this is the result!

<iframe width="560" height="315"
src="https://www.youtube.com/embed/XODomywp1lk" 
frameborder="0" 
allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" 
allowfullscreen></iframe>


How to make a simple people counter with unsupervised learning
--------------------------------------------------------------

My goal is to make a simple classifier that will classify frames into one of three categories:

1. Nothing happening
2. Someone entering a zone
3. Someone leaving a zone

I opted to use a neural networks approach, because it has a few valuable properties.

1. It's easy to express interesting unsupervised learning ideas as neural networks. I knew I didn't want to spend time labelling data, so unsupervised learning would definitely be part of my approach.
2. Pytorch supports Onnx export, so I knew I'd be able to run my code on the pi through a pi Onnx library.
3. If I needed to do any data cleanup, I knew that as long as it was expressed in Pytorch, it would be exported by Onnx.

In the end I used the following architecture, and trained in on a small dataset of me walking back and forth.

* A small autoencoder is trained on single frames from the AM8833
* The encoded frames have dimensionality 3-5
* The encoded frames were run through a K-means classifier, with three clusters. This corresponds to the three states above. Note that this was the **sklearn** kmeans implementation.
* I extracted the centroids from KMeans, and created a Pytorch version of a KMeans classifier (just the classifier, not the training)

Then, I put all this work end to end to make a single Pytorch module - that I could export with Onnx.


```python
# The autoencoder
decoder = nn.Sequential(
    nn.Linear(64, 32),
    nn.PReLU(),
    nn.Linear(32, 4),
)

encoder = nn.Sequential(
    nn.Linear(4, 16),
    nn.PReLU(),
    nn.Linear(16, 64),
)

net = nn.Sequential(
    decoder,
    encoder
)

```

```python
# The KMeans implementation
class KMeansLookup(nn.Module):
    def __init__(self, centroids, net):
        super().__init__()
        self.centroids = nn.Parameter(centroids)
        self.net = net
        
    def forward(self, x):
        cls_x = self.net(x)
        dists = []
        for ix, c in enumerate(self.centroids):
            dists.append(torch.norm(cls_x - c))
        dists = torch.stack(dists)
        return dists.argmin()
```

```python
# The export using Onnx
mdl = KMeansLookup(torch.tensor(cls.cluster_centers_).float(), decoder)
torch.onnx.export(mdl, x[0], 'test.onnx', verbose=True, training=False, example_outputs=r)
```

For all the code, check out the [notebook with all the code](https://gitlab.com/ririw/microprojects/-/blob/master/walker/Model%20training.ipynb)

Model outputs
-------------

To get a sense for how this "looks", I've extracted a few plots. The first is of the encoded frames from the autoencoder. 

![encoded outputs from the model](/assets/thermaltracker/encoded.png)

The spikes are me walking in and out of frame - you can see why it might be easy for KMeans to sort these into three clusters! In fact, this is what KMeans produces:

![KMeans](/assets/thermaltracker/kmeans.png)

So all I need to do now is run this model in my rust executable, and then transform the state transitions into lights changing!

How to compile a model into a rust executable
---------------------------------------------

This was a little bit fiddly - mostly because it takes a bit of work to get data from the internal format my program uses, into the format that Onnx expects. 
However, with some experimentation I was able to do so, and you can take a look [at the code](https://gitlab.com/ririw/microprojects/-/blob/e3dad6281d869720debcae6bf1dfd56e15df3842/walker/src/model.rs#L25-40) to see how I did it.

The other thing I **really** liked was that I could compile my model into the executable itself, rather than keeping it as a separate file. The rust `include_bytes` macro helped here. This keeps the deployment simple and easy!

The only other piece of significant work was making a small state machine to manage the lights as people move around.

Wrapup
------

I was quite happy with most of this process. The ML side needs some refinement, I think it still isn't robust enough. I'm very happy with how Onnx worked out, and I'll continue to push models into edge devices in this way in future projects.
