---
layout: post
title: "Simple backpropogation from scratch"
---
Most sizable neural networks today rely on backpropogation to make the training process more efficient.
While it is pretty ubiquitous in libraries such as [torch.autograd](https://pytorch.org/tutorials/beginner/blitz/autograd_tutorial.html), I was curious how difficult it would be to implement it myself without using existing autodiff libraries.
Turns out: not that difficult, and can be done with <100 lines of Python!

In this post, we'll derive backpropogation from scratch, implement it, and use it to train a simple neural network.
Crucially, for learning purposes, we won't be using libraries like Torch; rather, we will use only vanilla Python and numpy for matrix operations.
I've also tried to strike a balance between simplicity and depth, so that anyone with a passing understanding of matrices and derivatives should be able to follow along.

## Background
A neural network consists of a number of "neurons" connected via layers that work together to produce some output.
A simple three-layer, fully-connected neural network can be seen below.
<img src="/assets/images/nn.png" alt="diagram of 3-layer neural network" width="650"/>

*3-layer neural network*

In this example, we have two inputs, $$x_1$$ and $$x_2$$, that go through three layers before producing the final output, consisting of $$y_1$$ and $$y_2$$.
Each layer $$l$$ of neurons:
1. takes some input from the previous layer $$l-1$$
2. transforms it by multiplying the input by a weight matrix $$W_l$$ and adding a bias term $$b_l$$
3. applies an activation function $$\sigma$$
4. then finally, passes the output to the next layer

The training process, then, consists of finding the optimal weights and biases in the network such that some cost function is minimized.
The cost function can be seen as how poorly the network performs, so lower is better.
The hope is that given enough training data and iterations, we can reduce the cost function to a minimum, which by extension means the network will do a pretty good job of predicting an output.

Below is some notation to formalize this idea.

* $$X$$: input set, consisting of $$x_1, x_2, ... , x_n$$
* $$W_l$$: weight matrix of layer $$l$$ where $$w^l_{ij}$$ is the weight of the connection between the $$i$$th neuron of layer $$l$$ and the $$j$$th neuron of layer $$l-1$$. This is a matrix with size $$n x m$$ where $$n$$ is the number of neurons in the current layer and $$m$$ is the number of inputs to the current layer.
* $$z_l$$: the weighted sum of all inputs into layer $$l$$. This is a vector with number of rows = number of neurons in layer $$l$$.
* $$a_l$$: the activation of layer $$l$$, equal to $$\sigma(z_l)$$. This is a vector with number of rows = number of neurons in layer $$l$$.

But how do we adjust the weights and biases?
A common method of doing this is via gradient descent.
Since we want to minimize some cost function, gradient descent essentally finds the gradient of the function at some point, then moves the weights and biases in the opposite direction of the gradient.
<img src="/assets/images/gradient.png" alt="diagram of 3-layer neural network" width="500"/>

*Finding local minima via gradient descent*


How to _efficiently_ find the gradient of the cost function with respect to each of the input weights and biases is where backpropogation comes into play.

## Derivation
For each layer $$l$$, we want to find the gradient of the cost function w.r.t. its weights, or: 
$$\begin{align}\frac{\partial C}{\partial W_l}\end{align}$$.
The key ingredient here is chain rule of calculus.
That is, if $$z = f(y)$$ and $$y = f(x)$$, then:
$$
\begin{align}
\frac{\partial z}{\partial x} &= \frac{\partial z}{\partial y} \frac{\partial y}{\partial x}.\\
\end{align}
$$

### Weight updates for output layer
Recall the relationship between $$W_l$$, $$z_l$$, $$a_l$$, and $$C$$.
$$W_l$$ is multiplied with the output activation from the previous layer $$a_{l-1}$$ to form $$z_l$$.
$$z_l$$ is passed into our activation function $$\sigma$$ to form $$a_l$$.
This process is repeated until the final activation $$a_L$$ is passed to the cost function $$C$$.

$$\begin{align}
z_l &= W_la_{l-1}\\
a_l &= \sigma(z_l)\\
C(a_L) &= \sum_{i=1}^{n}(a_L - y_i)^2\\
\end{align}$$

Deriving $$\frac{\partial C}{\partial W_L}$$ for output layer is thus:
$$
\begin{align}
  \tag{1.1}
  \frac{\partial C}{\partial W_L} &= \frac{\partial C}{\partial a_L}\frac{\partial a_L}{\partial z_L}\frac{\partial z_L}{\partial w_L}\\
  &= \frac{\partial}{\partial a_L}(a_L - y)^2 \odot \sigma'(z_L) \frac{\partial z_L}{\partial W_L}\\
\end{align}
$$

Here we can simplify our notation by defining a new value, $$\delta_l$$, as the partial derivative of the cost in terms of our weighted input $$z_l$$:

$$
\begin{align}
\delta_l &= \frac{\partial C}{\partial z_L}\\
\end{align}
$$

So then we get:

$$
\begin{align}
  \frac{\partial C}{\partial W_l} &= \delta_L\frac{\partial z_L}{\partial W_L}\\
  &= \delta_L\frac{\partial (W_La_{L-1})}{\partial W_L}\\
  &= \delta_L(a_{L-1})^T
\end{align}
$$

Intuitively, $$\delta_l$$ represents the "error," or how sensitive the output of the cost function is in terms of the current layer's weighted input $$z_l$$.
If this value is large, that means the cost can be significantly reduced given a small change in the weighted input, so the weighted input $$z_l$$ is pretty far off from the desired value.
Conversely, a small $$\delta$$ means we can't reduce the cost much more, thus it is close to the desired value.

### Weight updates for hidden layers
For the hidden layers, the relationship between the weights and the final cost is a bit more implicit.
But essentially it is a repeated application of the chain rule from some $$W_l$$, through the network, and ending at the cost function.

$$
\begin{align}
  \tag{1.2}
  \frac{\partial C}{\partial W_l} &= \frac{\partial C}{\partial a_L}\frac{\partial a_L}{\partial z_L}\frac{\partial z_L}{\partial W_L}\frac{\partial W_L}{\partial a_{L-1}}\frac{\partial a_{L-1}}{\partial z_{L-1}}\frac{\partial z_{L-1}}{\partial W_{L-1}} ... \frac{\partial a_{l+1}}{\partial z_{l+1}}\frac{\partial z_{l+1}}{\partial W_{l+1}}\frac{\partial W_{l+1}}{\partial a_l}\frac{\partial a_l}{\partial z_l}\frac{\partial z_l}{\partial W_l}\\
  &= \delta_{l+1}\frac{\partial z_{l+1}}{\partial W_{l+1}}\frac{\partial W_{l+1}}{\partial a_l}\frac{\partial a_l}{\partial z_l}\frac{\partial z_l}{\partial W_l}\\
  &= W_l^T\delta_{l+1}\frac{\partial a_l}{\partial z_l}\frac{\partial z_l}{\partial W_l}\\
  &= W_l^T\delta_{l+1}\frac{\partial \sigma(W_l a_{l-1})}{\partial z_l}\frac{\partial z_l}{\partial W_l}\\
  &= W_l^T\delta_{l+1}\odot \sigma'(W_la_{l-1}) \frac{\partial z_l}{\partial W_l}\\
  &= \delta_l \frac{\partial z_l}{\partial W_l}\\
  &= \delta_l \frac{\partial (W_La_{l-1})}{\partial W_l}\\
  &= \delta_l (a_{l-1})^T\\
\end{align}
$$

But most importantly, we see that the error term $$\delta_l$$ depends on that of the next layer $$\delta_{l+1}$$.
So instead of iterating through the entire network each time for every layer, we can start at the output layer and move backward, saving the value each time to be used by the previous layer on the next iteration.
In algorithm design, this technique is known as dynamic programming.

More formally, the error term for the output layer is

$$
\begin{align}
\delta_L = (x_l - y)\odot \sigma'(W_La_{L-1})
\end{align}
$$

while the error for every other hidden layer is

$$
\begin{align}
\delta_l = (W_{i+1})^T\delta_{i+1}\odot \sigma'(W_la_{l-1}).
\end{align}
$$

where the derivative of the sigmoid function is conveniently:

$$
\begin{align}
\sigma'(x) &= x(1 - x)\\
\end{align}
$$

### Bias updates
Using the above derivation, finding the bias update is much simpler.

$$
\begin{align}
\frac{\partial C}{\partial b_l} &= \frac{\partial C}{\partial z_l}\frac{\partial z_l}{\partial b_l}
&= \delta_l \frac{\partial(W_la_{l-1} + b_l)}{\partial b_l}
&= \delta_l
\end{align}
$$

### Summary of steps
Now that we have all the pieces, we have a formula for training our network using backpropogation!
To summarize:
1. For each training sample, forward the input through the network and arrive at a cost.
2. To reduce that cost, we backpropogate the error term from the last layer to the first layer. Each time, we use the error to update the weights at each layer, i.e. 
$$
W_l = W_l - \alpha\frac{\partial C}{\partial W_l}
$$
where
$$ \frac{\partial C}{\partial W_l} = \delta_L(a_{L-1})^T$$ when $$l = L$$ and $$\frac{\partial C}{\partial W_l} = \delta_l(a_{l-1})^T$$ if otherwise. For the sigmoid function, $$\delta_L = (x_l - y)\odot W_La_{L-1}(1 - W_La_{L-1})$$ for the output layer and $$\delta_l = (W_{i+1})^T\delta_{i+1}\odot W_la_{l-1}(1 - W_la_{l-1})$$ for each hidden layer $$l$$.
3. Similarly, we also update the bias term via $$b_l = b_l - \alpha \delta_l$$.
4. Repeat the above steps for a large number of iterations until the cost is sufficiently small.