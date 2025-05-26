---
layout: post
title: "Attention as a kernel smoothing problem"
---

[Reddit discussion](https://www.reddit.com/r/MachineLearning/comments/1kuoifv/r_attention_as_a_kernel_smoothing_problem/)

This post is about a rarely discussed interpretation of "attention," the underlying technique behind transformers, which has found its way into everything from natural language processing to computer vision.
Namely: _attention can be viewed as a kernel smoothing problem for estimating a function given surrounding observations._

This is by no means an original observation, and has already been covered in, for example, [Tsai et. al](https://arxiv.org/abs/1908.11775) or [Song et. al](https://arxiv.org/abs/2006.06147). But it is still surprisingly absent from any material you can find online explaining this subject. More than anything, I think viewing attention through this lens gave me the clearest conceptual understanding of what attention is really doing.

## Attention
To recap, the idea of attention is as follows.
We have an input sequence of vectors $$<x_1, x_2, ...,x_n>$$ and an output sequence of vectors $$<y_1, y_2, ..., y_m>$$, and we want these vectors to somehow encode information about one another.
Perhaps we're doing machine translation, where the input sequence represents words in English, and the output sequence represents words in Korean.
We want the English and Korean words to somehow absorb context from surrounding words in their sequence, a.k.a. "self attention." We also the Korean words to derive meaning from English words across the pond, referred to as "cross attention."

We can formulate this as a deep learning problem. To do this, we divide this process into two stages: the encoder, which processes the input sequence, and the decoder, which processes the output sequence, each with its own learnable weight parameters. This is then repeated a large number of times until some cost function is sufficiently small, at which point each vector will have learned its surrounding context.

The way we get these vectors to "talk" to each other is by projecting each vector onto three separate subspaces; these new vectors are known as the "query," the "key," and the "value."

This projection is done by left-multiplying the original vectors $$x_1, ... , x_n$$ by learnable weight matrices, $$W_Q$$, $$W_K$$, and $$W_V$$.

$$\begin{align}
q_i &= x_{Qi}W_Q\\
k_i &= x_{Ki}W_K\\
v_i &= x_{Vi}W_V\\
\end{align}$$

where $$x_{Qi} = x_{Ki} = x_{Vi}$$ for self-attention. In cross-attention, $$x_Q$$ comes from the decoder side while $$x_K$$ and $$x_V$$ comes from the encoder side.

One way to intuit this is to view the query as a database query, and the keys and values as the entries stored in the database.
We can match the query with each key and return a similarity measure where higher means a "better match." Then, rather than returning a discrete set of results that "match," we instead return a _distribution_ of results, where each result is weighed by the similarity measure. That is, $$similarity(query, key) \times value$$.

To formalize this: for each query $$q_i$$, key $$k_i$$ and value $$v_i$$, we define attention as

$$\begin{align}
\tag{1.1}
attention(q_i, k_i, v_i) = softmax(\frac{q_ik_i^T}{\sqrt{d_k}})v_i
\end{align}$$

where $$d_k$$ is some stabalization term needed for numerical stability.

To calculate how well our query associates with _all_ possible key/value pairs, we can pack all the keys into a matrix $$K$$ and the values into a matrix $$V$$.
Then, we get a condensed representation

$$\begin{align}
\tag{1.2}
attention(q_i, K, V) = softmax(\frac{q_iK^T}{\sqrt{d_k}})V
\end{align}$$

The result of the softmax is a probability distribution where each column $$j$$ measures the interaction between $$q_i$$ and $$k_j$$. This is used to weigh each value $$v_j$$.

### Multi-headed attention
This, however, only uses one set of learnable weight parameters, namely $$(W_Q, W_K, W_V)$$.
Ideally, we want to train _multiple_ weights for each query, key and value so that they can all influence each other in different ways.
This will give us a set of $$h$$ queries, keys, and values.

$$\begin{align}
Q_i &= W_{Qi}Q\\
K_i &= W_{Ki}K\\
V_i &= W_{Vi}V\\
\end{align}$$


for $$i = 1, 2, ..., h$$. With $$h$$ queries, keys, and vectors, we can then calculate $$h$$ attention heads:

$$\begin{align}
head_i = attention(Q_i, K_i, V_i)
\end{align}$$

Usually the dimension of the new weights will be some fraction of the dimension of our original vectors, for example $$d_{model} / h$$.
We can then stack them all together horizontally:

$$\begin{align}
attention = horzcat(head_1, ..., head_h)W_O
\end{align}$$

(Don't worry about what $$W_O$$ is for now; it isn't particularly important.)

Since each head is of dimension $$n \times d_{model} / h$$, stacking them horizontally gives us a matrix of dimension $$n \times d_{model}$$, equivalent to our previous single-headed attention.
The benefit of using multiple, smaller weight matrices instead of a single large one is that it lets us capture a more diverse set of interactions.

This is the basics of the so-called attention mechanism popularized by the 2017 paper [_Attention Is All You Need_](https://arxiv.org/abs/1706.03762).

## Kernel smoothing

Suppose we have a set of points $$(x_1, y_1), (x_2, y_2), ..., (x_n, y_n)$$ and an observation $$x_o$$, and we want to predict what the corresponding $$y$$ would be, i.e. $$\hat{y}(x_o)$$

One way to do this is to identify the $$k$$ closest points to $$x_o$$, i.e. $$x^\prime_1,...,x^\prime_k$$ and average up their $$y$$ values. More formally:

$$
\hat{y}(x_o) = \frac{1}{k} \sum_{i=1}^{k}{y^{\prime}_i}
$$

But this doesn't take into account the _closeness_ of each $$x^\prime_i$$ to $$x_o$$. Thus, we introduce a similarity measure $$K(x_1, x_2)$$ that is maximized when $$x_1 = x_2$$ and use it to weigh each point. This similarity measure is known as the kernel, and there are many ways to define it, each with different tradeoffs. Our expression then becomes:

$$
\tag{2}
\hat{y}(x_o) = \sum_{i=1}^{n} \frac{K(x_o, x_i)y_i}{\sum_{i=1}^{n}{K(x_o, x_i)}} $$

Note that we no longer have to pick out $$k$$ points; we can just use all $$n$$ points and the similarity function should take care of eliminating unrelated points by decaying their weights towards zero.

## Attention as a kernel smoother
What does kernel smoothing have to do with attention?
It turns out that with a little derivation, the attention mechanism can be reformulated as a kernel smoothing problem!

Let's revisit an attention interaction between a single query, key and value.

$$\begin{align}
attention(q_i, k_j, v_j) &= softmax(\frac{q_ik_j^T}{\sqrt{d_k}})v_j\\
                   &= \frac{exp(q_ik_j^T/\sqrt{d_k})}{\sum_{j=1}^nexp(q_ik_j^T/\sqrt{d_k})}v_j\\
\end{align}$$

Letting $$K(q, k) = exp(qk^T / \sqrt{d_k})$$ we get

$$\begin{align}
\tag{3}
attention(q_i, k_j, v_j) &= \frac{K(q_i, k_j)v_j}{\sum_{l=1}^{n}{K(q_i, k_l)}}
\end{align}$$

Observe that this takes on a similar form to the kernel smoothing equation in (2)!
In particular, the "query" is the $$x_o$$, the "keys" are the $$x_i$$'s and the "values" are the $$y_i$$'s (albeit with an additional linear transformation). And lastly, the exponential function of the query-key dot product divided by the stabalization term is the kernel.

With this realization, multi-headed attention with $$h$$ heads can be reduced to combining $$h$$ different kernel smoothers!

One final thing to point out is that our specific kernel is an example of a non-symmetric kernel, i.e. $$K(a, b) \ne K(b, a)$$, due to the fact that we apply different linear mappings for the queries and for the keys.
This can be more clearly seen if rewritten as a function of the input vectors $$x$$:

$$\begin{align}
K(x_q, x_k) = exp((x_qW_Q)(x_kW_K) / \sqrt{d_k})
\end{align}$$

With a bit of thought, this should make sense, since the ways two words influence each other in a sentence shouldn't necessarily be identical.


## Summary
In summary, attention can be seen as a kind of kernel smoother with a special non-symmetric kernel.
The way we get a word to absorb contexual information from its neighboring words is by calculating its interaction with each neighbor and then weighing it by a relevance score, akin to how kernel smoothing approximates a function at a given point by calculating a weighted average of its surrounding points.
