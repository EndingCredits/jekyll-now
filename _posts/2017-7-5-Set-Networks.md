---
layout: post
title: Set Networks - Another tool for your toolbox
---

Currently WIP, all feedback welcome.

>*Q*: What do recent teqniques for [neural machine translation](https://arxiv.org/abs/1706.03762), [relational reasoning](https://arxiv.org/abs/1612.00222), and [learning on point-clouds](https://arxiv.org/abs/1612.00593) have in common?
>
>*A*: they all use set networks!

A deep-learningomancer's toolkit is seemingly endless. Simple spells, such as batch-normalisation and skip connections are joined by arcane magic such as  (not to mention the six hundred and sixty six different species of GAN).

Needless to say, the number of different model architectures is also collosal.
However, broadly speaking, we can divide most commonly used deep learning models into one of three categories: those designed to operate on feature vectors (*e.g.* fully connected), those designed to operate on tensor-structured data such as images, or audio (*e.g.* CNNs), and those designed to work on sequential data (*e.g.* RNNs).

While this is far from a comprehensive classification of all models (doubtless large numbers of networks exist for a whole host of different esoteric input formats, not to mention the dreaded [chimera](https://arxiv.org/abs/1706.05137)), but it does seem to feature one glaring ommision: models designed to operate on sets. Enter the set network!

<!-- Why sets?
       Appear in lots of places.
       (Seq2Seq for sets)
       Alternative to sequences.
       Multi-instance learning.
     -->
<!-- Why this post?
       Used in lots of places, but no overarching consensus.
       Even closely related papers miss this link.
         Papers don't mention trying different layer types
       Many places where a set network is an obvious 
       Yeilding SOTA results in certain domains.
     -->
<!-- What this post is?
       Definition of set network
       Summary from other papers (in particular deep sets).
       Attempt to unify different ideas.
       Own notes which may be useful to others.
       Code (in python and tf) so other people can get started.
     -->
<!-- Table of contents -->

<!-- Feedback, notes, and acknowledgements -->

### The simple set network
<!-- Definition of set network: equivariance and symmetry -->
Just as CNNs operate over fixed-size vectors arranged into a grid pattern, and RNNs operate over fixed-size vectors arranged into a sequence, we are interested in models that can operate over fixed-size vectors organised a set.

Unfortunately we're working with *tensorflow*, not *setflow* so we can't simply input a set into our network. We can get around this by representing our set as a tensor: Given a set of vectors `{v1, v2, ..., vn}` we can easily turn this into a single tensor by concatenating the vectors along an extra *0*th dimension. Note (and this will be important later) that a given set can have multiple representations since we can permute elements of a set without changing it, hence any permutation along the *0*th axis of our tensor is a representation of the same set.

>If our set is given as a python list, `set = [v1, v2, ..., vn]`, we can use the tersorflow operation `tensor = tf.pack(set)`. However, in practice, it's easier to do this in either python or numpy as part of the data-preparation process, and feed tensorflow a single tensor.

#### Pooling

Let's assume for now that we want our model to produce a single fixed-size output vector, *i.e.* our model is a function <!-- f_k: X^n \rightarrow \mathbf Y, where X = \mathbf R^m} and Y = \mathbf R^{D_{OUT}} --> (in fact, we really need a sequence of functions <!-- \{ f: X^k \rightarrow \mathbf Y \}_{k \in \mathbf N} --> as we want to handle sets with different lengths k). We want the output to be invariant to different representations of the same the input set. Hence, we are looking for symmetric functions of vectors.

Fortunately some very simple candidates for these already exist, including most *tensorflow* ops beginning with `tf.reduce_`.
By reucing along the *0*th axis these operations transform an n times m dimensional tensor into a single m-dimensional vector.
For now we will only consider max pooling, *i.e.* `tf.reduce_max`, but most of the following will also apply to alternatives such as `tf.reduce_sum`, and `tf.reduce_sum`.

<!--  Notes on how to compose from simple arithmentic functions.
      In fact we can devise such an function using any associative symmetric operation. -->
      
<!-- Pooling operations are the bread of set networks, they are what enable to turn set into vector -->

However, naively applying max pooling to our input tensor is unlikely to give us good results. For example, if our input is a set of points (a point-cloud), then max pooling will give us one half of the bounding box of all points. While this is useful information, it doesn't tell us anything about the structure of the points. We need to first transorm our input points where max pooling preserves more information about the structure of the set. Additionally, max pooling doesn't have any trainable parameters, so we need an additional trainable component to turn our architecture into a trainable model.

<!-- A note on masking -->
>An important note is how to handle sets of different numbers of elements. This is not a problem if we are only working with one set at a time as we can set the *0*th axis to be dynamically sized, but when we want to batch multiple input tensors into a single batch tensor, this can cause problems as the inputs will have different dimensions. 
>The way around this is to pad all input tensors to a given size by using dummy input elements. It's also useful to also use a  mask to indicate which elements are real, and which ones are dummies, to ensure the dummy elements don't iterfere with the real ones under certain operations.

#### Element embeddings

For each of our input vectors, we would like to transorm it in some way to work better with max pooling. We could do this via some static operation, but a much better alternative is to use some trainable function. A nice convenient function is a fully connceted neural network with *m* input neurons and *d* output neurons. Given a network `net(x)` and input tensor `<v1, v2, ..., vn>`, we can apply this network form a new tensor `<net(v1), net(v2), ..., net(vn)>`. We'll call such an operation 'set transformation by `net()`'.

<!-- Embedding is the butter (bread by itself is technically functional, but not very nice) -->

> We can view this network set transformation as a single network with a weight-sharing scheme (in fact, it turns out that this is the same as applying a sequence of 1-dimensional convolution filters). By doing this we see that we can train this architecture using standard backpropogation and gradient descent. 
> A more comprehensive explanation (as well as some additional information on deep set networks) can be found in [this paper](https://arxiv.org/abs/1611.04500).

We can implement a linear set transormation layer in tensorflow via `layer_out = tf.map_fn(lambda x: tf. matmul(x, w) + b), layer_in)`, however, it's better (for hardware optimisation) to use a convolution operation. These layers can be stacked (along with activation functions) to create a full network set transformation.
<!-- `layer_out = tf.nn.conv1d(layer_in, [W_1], stride=1, padding="SAME") + b` -->

Another way of looking at this is saying we would like some class of functions of the form <!-- \{ f_k: X^k \rightarrow \times \theta \mathbf Y^k \}_{k \in \mathbf N} --> where a change in permutation to the inputs is matched by a corresponding change in permutation to the outputs, *i.e.* each <!-- f_k --> is equivariant (excluding the parameter-space). This encompasses a greater space of possible functions than applying a simple transfomration to each vector, as it allows the transformation of each vector to depend on other vectors in the set. We'll call this larget category 'set operations' where set transofrmations are a subset. These will be useful later when we discuss deep set networks, and self-attention.


#### Putting everything together

We can now combine this set operation with our pooling operation to obtain a set representation. While we could use this set representation directly, personal experience suggests that it's better to feed this representation into a final fully connected network (presumably as this allows the set operation to focus on preserving as much information as possible when pooled).

<!-- A note on why max pooling -->

<!-- Experiment code & instructions -->
<!-- Example set tranformation layer -->


### Deep set networks

While tech technique described above is surprisingly powerful, it's still rather primitive: Elements are naiveley embedded into a higher dimensional space and then pooled to get a single representation. <!-- This can (and often does) give perfectly good results on a variety of task, however it can prove brittle in certain situation, for example, on point clouds with extreme variances in scale. -->

<!-- While the set networks above may be deep in terms of a number of individual layers, they are shallow in that they only produce a single set representation. We would like networks that produce multiple sucessively more refined set representations, *i.e.* deep set networks. -->


#### Enhanced element embeddings

An obvious potential enhancement to our architecture would be to use some statistics about the set to help inform the transorfmation process. For example, we could first normalise our inputs by dividing by the standard deviation of the set and subtracting the mean. This is a step in the right direction, however, rather than fixed statistics and operations, it would be better if we could enable our architecture to learn these by itself.

There are many ways to do this, but for now we will look at at simple extension to our set transormfarion. For this we need a netwrok which takes two inputs, `net(x,y)`. Now, given an input tensor `<v1, v2, ..., vn>` and a set statistic `s`, we can get a new tensor `<net(v1,s), net(v2,s), ..., net(vn,s)>` which can use this set-statistic to alter the transormfation. We call this type of operation a 'contextual set transformation'. contextual set transformations are a subset of set operations, and a superset of set transformations.

There are many different choices for `net(x,y)`, but we will consider the simple case where `net(x,y)` is a fully connected neural network with `dim(x)+dim(y)` inputs and *d* outputs, where `x` and `y` are concatenated and the resulting vector is fed into the network.

<!-- Example set tranformation layer -->

But where can we get this set statistic from? We could use some generic statistic of the set (*e.g.* mean, or standard deviation), but we've already discussed a general method for representing sets with vectors: via set networks! 

> While the set operations are useful for creating better embedding sto pass through pooling, there are many cases where individual element embeddings can used by themselves. For example, for semantic segmentation, the embedding of each element could correspond to the networks predicted class of that element. For simple set networks, these individual element embeddings are not too interesting by themselves, as they were produced without additional information from other elements in the set, however for deep set networks they are a lot more useful. 


#### Building deep set networks

<!-- Defn of stages -->
<!-- Notes on architecture styles -->

<!-- Experiment code & instructions -->


#### Other layer types

This section will cover two alternatives to the general set transofmration layer: the set layer used by [Ravanbakhsh et al.](https://arxiv.org/abs/1611.04500) and the T-net of [Qi et al.](https://arxiv.org/abs/1612.00593).


<!-- ### Summary -->


### Other techniques

#### Self attention

This section will cover self-attention as used in Deepmind's All You Need is Attention paper

#### Heirarchical set networks

This section will cover applying set-networks to sets of sets (by partitioning sets) as usied in [Pointnet++](https://arxiv.org/abs/1706.02413) and other places.

#### Different element lengths

This section will cover some of my notes on dealing with sets which consist of vectors of different lengths.