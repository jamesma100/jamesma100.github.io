---
layout: post
title: "Simple backpropogation in Python and numpy"
---
In our [previous post](/2024/deriving-backpropogation/), we introduced backpropogation as a means for efficiently training neural networks.
Here we'll attempt to implement a simple Python framework to train a fully-connected neural network given some training data and a description of the network architecture.
You can find the full code [here](https://github.com/jamesma100/nn.learn/blob/main/backprop.py).

The goal here is _not_ to give a primer on neural networks and how to best design them. Rather, we will focus on the training process via the backpropogation algorithm we derived last time. 

Once again, for learning purposes, we won't be using pytorch and friends.
We will use only Python and numpy (for its support for matrices).

For the full details I encourage reading the previous post. But in a dainty nutshell, this is what we are trying to do.
- We have a neural network that takes in some input, forwards it through multiple layers of weights and biases, and spits out an output.
- We want to find the _optimal_ weights and biases to minimize some cost function, which means to find the gradient of the cost with respect to each parameter.
- Rather than compute the full gradients directly, we'll calculate them recursively, starting at the last layer and ending at the first.
- Once we have each gradient, we update the weights and biases by moving in the opposite direction of the gradient.
- We repeat until our cost is sufficiently small.

In this example, we'll try to predict 4 bit binary addition. For the sake of simplicity, we'll avoid operations that lead to overflow in our training set.

```python
DATA_IN = [
    np.array(
        [[int(b)] for b in format(i, "04b")] + [[int(b)] for b in format(j, "04b")],
        dtype=np.float64,
    )
    for i in range(8)
    for j in range(8)
]
DATA_OUT = [
    np.array([[int(b)] for b in format(i + j, "04b")], dtype=np.float64)
    for i in range(8)
    for j in range(8)
]
DATA_ZIP = list(zip(DATA_IN, DATA_OUT))
random.shuffle(DATA_ZIP)
DATA_IN, DATA_OUT = zip(*DATA_ZIP)
```

We have 8 x 8 = 64 training examples, so we'll use 45 for training and 19 for testing, corresponding to an approxmiate 70/30 split.
```python
TRAIN_IN, TRAIN_OUT = DATA_IN[:45], DATA_OUT[:45]
TEST_IN, TEST_OUT = DATA_IN[45:], DATA_OUT[45:]
```

We will try to implement this network class defined below, which will encapsulate the entirety of our neural network, including setup, initialisation, and training.
```python
class Network:
    def __init__(self, inputs_len, arch, num_iter):
        self.inputs_len = inputs_len
        self.arch = arch
        self.rng = np.random.default_rng()
        self.weights = []
        self.biases = []

        self.num_iter = num_iter
        self.rate = 0.1  # learning rate
```
The `Network` class takes in the number of inputs, the architecture, and the number of iterations to train for.
The `arch` parameter is a list consisting of the number of neurons in each layer. For instance, a network with 2 inputs, 3 hidden layers of 4 neurons each, and one output layer can be represented as:
```python
nn = Network(2, [4, 4, 4, 1])
```
The `weights` and `biases` lists will each contain `n` elements where `n` is the number of layers. These parameters will be constantly adjusted throughout the duration of the training process.

To start, we'll simply randomise our weights and biases, making sure our matrix sizes line up for left-multiplication, since our final output is defined as $$\begin{align}a_L = W_LW_{L-1}...W_2W_1x\end{align}$$.
```python
def randomise(self):
    num_cols = self.inputs_len
    for num_rows in self.arch:
        weight = self.rng.random((num_rows, num_cols))
        bias = self.rng.random((num_rows, 1))
        self.weights.append(weight)
        self.biases.append(bias)
        num_cols = weight.shape[0]
```
Then, to complete a single forward pass, we simply left-multiply the weights and biases that we've stored.
```python
def forward(self, inputs) -> list[np.array]:
    activations = []
    activation = inputs
    n = len(self.weights)
    for i in range(n):
        activation = sigmoid(self.weights[i]@activation + self.biases[i])
        activations.append(activation)
    return activations
```
A rather important piece of implementation detail is that we're not only returning the final activation layer; we are returning a _list_ of activations, one per layer, since we need these intermediate values for backpropogation.

After each forward pass, we need a corresponding backward pass, during which we backpropogate our error term and save those intermediate values as well.
In our previous post, we derived the error terms using some linear algebra. They are:
- $$\begin{align}\delta_L = (a_L - y)\odot W_La_{L-1}(1 - W_La_{L-1})\end{align}$$ for the output layer $$L$$;
- $$\begin{align}\delta_l = (W_{l+1})^T\delta_{l+1}\odot W_la_{l-1}(1 - W_la_{l-1})\end{align}$$ for each hidden layer $$l$$.

```python
def backward(self, output, activations) -> list[np.array]:
    # error of output layer

    err_out = activations[-1] * (1 - activations[-1]) * (activations[-1] - output)
    n = len(activations)
    errors = [np.array([]) for i in range(n)]
    errors[-1] = np.array(err_out)

    # error of hidden layers

    for i in range(n - 2, -1, -1):
        err_hidden = (
            activations[i]
            * (1 - activations[i])
            * (self.weights[i + 1].transpose() @ errors[i + 1])
        )
        errors[i] = err_hidden
    return errors
```

Now that we have all the key pieces, we can begin training the network. Recall a single iteration of the training process.
- split our input data into multiple batches
- for each batch:
    - for each sample in batch:
        - call `forward()` on the input and get a list of `n` activations
        - call `backward()` on the output and get a list of `n` errors
    - for each layer $$l$$:
        - calculate weight and bias changes
        - adjust the weight and bias terms

The meat of the training lies in the following two functions.
First, `process_batch` takes a batch of training samples, and for each sample, performs a forward pass and a backward pass while saving the intermediate activations and errors.
We'll return a tuple consisting of 1. the activations 2. the errors and 3. the cost of the batch.
```python
def process_batch(self, batch) -> (list[list[np.array]], list[list[np.array]], np.float64):
    all_activations: list[list[np.array]] = []
    all_errors: list[list[np.array]] = []
    total_cost_of_batch = 0
    batch_in, batch_out = batch
    num_samples_in_batch = len(batch_in)

    for i in range(num_samples_in_batch):
        activations = self.forward(batch_in[i])
        all_activations.append(activations)
        errors = self.backward(batch_out[i], activations)
        all_errors.append(errors)
        local_cost = self.cost(activations[-1], batch_out[i])
        total_cost_of_batch += local_cost
    return (all_activations, all_errors, total_cost_of_batch[0])
```
The second main function, `update_params`, takes the cached activations and errors we just found for our batch and adjusts the weights and biases.
```python
def update_params(self, all_activations, all_errors, train_in, num_samples_in_batch):
    num_updates = len(self.weights)
    for i in range(num_updates):
        if i != 0:
            delta_w = sum(
                [
                    errors[i] @ activations[i - 1].transpose()
                    for activations, errors in zip(all_activations, all_errors)
                ]
            )
            self.weights[i] -= self.rate / num_samples_in_batch * delta_w
        else:
            delta_w = sum(
                [
                    errors[i] @ sample.transpose()
                    for sample, errors in zip(train_in, all_errors)
                ]
            )
            self.weights[i] -= self.rate / num_samples_in_batch * delta_w
        delta_b = sum([errors[i] for errors in all_errors])
        self.biases[i] -= self.rate / num_samples_in_batch * delta_b
```

We can now initialise our network and begin training it.
I picked a random architecture consisting of two hidden layers, each with 12 neurons.
Our input size is 8 since we have two 4-bit integers.
```python
nn = Network(8, [12,12,4], 100_000)
nn.randomise()
nn.train(TRAIN_IN, TRAIN_OUT)
```
This should take a while, since we are using the infamously slow Python$$^{TM}$$, but we can see the cost function slowly decreasing over time.
Notice that the cost is not strictly decreasing. It's possible for the gradient descent step to fall into a local minimum then climb back out. It's also likely that, since our training operates on batches, we found parameters that decreased the cost of the batch but ended up increasing the _overall_ cost across the entire training set. 
<img src="/assets/images/anim.gif" alt="animation of improving cost function" width="650"/>

You can adjust the learning rate `self.rate`, size of each batch, and the number of iterations to see how it affects the cost.

Now that our network has found the optimal weights and biases, we can see how well it performs.
Our network does great on training data:
```
6 + 5 = 11
2 + 4 = 6
1 + 6 = 7
7 + 1 = 8
4 + 3 = 7
1 + 0 = 1
0 + 4 = 4
5 + 3 = 8
1 + 4 = 5
7 + 6 = 13
```
But fails miserably on test data, which is perhaps unsurprising.
```
2 + 5 = 10
6 + 3 = 9
1 + 5 = 8
6 + 1 = 0
5 + 6 = 9
5 + 0 = 13
4 + 1 = 0
7 + 4 = 13
5 + 4 = 13
2 + 2 = 6
```
To address this problem of overfitting, one would usually use a regularisation term, such as a L1 or L2 norm of the weights. This will penalise large parameters and drive our solution toward simpler, sparser solutions.

So far, if you're unimpressed by our little neural network, I assure you it can do a lot more than add numbers.
It can also subtract numbers. You could even teach it how to convert between Fahrenheit and Celsius.

---
We glossed over a world of details to get to the heart of backpropogation quickly. For instance, how do we batch our training data? What should our cost function be? Should we be using sigmoid or RELU or something else? Each of these topics rightfully deserves its own branch of study.

Hopefully now you more or less understand how a neural network is trained, and more importantly, how backpropogation speeds up the process by avoiding naive recomputation of the gradient.
