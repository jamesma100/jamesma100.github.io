---
layout: post
title: "A gallery of randomly generated art"
---

In a [paper](https://users.ece.cmu.edu/~adrian/projects/validation/validation.pdf) from CMU, security researchers describe a nifty algorithm used to generate random images known as _RandomArt_.
In a nutshell, the algorithm takes as input a seed to a pseudo-random number generator and uses it to construct an [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) by selecting rules from a predefined grammar.
The AST describes a function mapping (x, y) pixel values to RGB values.

Recently I [implemented a version](https://github.com/jamesma100/randomart) of it in Haskell, and this page shows some images I was able to generate using a pretty simple grammar. Each image uses a different initial seed, so you can see that altering the seed results in a wildly different image, making it suitable for quickly spotting changes in, for instance, SSH public keys.

See the [appendix](#appendix) for more details about how this works and how to generate your own images.

<br>
<hr>

<img src="/assets/images/randomart/1.png" alt="" width="220"/>
<img src="/assets/images/randomart/2.png" alt="" width="220"/>
<img src="/assets/images/randomart/3.png" alt="" width="220"/>
<img src="/assets/images/randomart/4.png" alt="" width="220"/>
<img src="/assets/images/randomart/5.png" alt="" width="220"/>
<img src="/assets/images/randomart/6.png" alt="" width="220"/>
<img src="/assets/images/randomart/7.png" alt="" width="220"/>
<img src="/assets/images/randomart/8.png" alt="" width="220"/>
<img src="/assets/images/randomart/9.png" alt="" width="220"/>
<img src="/assets/images/randomart/10.png" alt="" width="220"/>
<img src="/assets/images/randomart/11.png" alt="" width="220"/>
<img src="/assets/images/randomart/12.png" alt="" width="220"/>
<img src="/assets/images/randomart/13.png" alt="" width="220"/>
<img src="/assets/images/randomart/14.png" alt="" width="220"/>
<img src="/assets/images/randomart/15.png" alt="" width="220"/>
<img src="/assets/images/randomart/16.png" alt="" width="220"/>
<img src="/assets/images/randomart/17.png" alt="" width="220"/>
<img src="/assets/images/randomart/18.png" alt="" width="220"/>
<img src="/assets/images/randomart/19.png" alt="" width="220"/>
<img src="/assets/images/randomart/20.png" alt="" width="220"/>
<img src="/assets/images/randomart/21.png" alt="" width="220"/>
<img src="/assets/images/randomart/22.png" alt="" width="220"/>
<img src="/assets/images/randomart/23.png" alt="" width="220"/>
<img src="/assets/images/randomart/24.png" alt="" width="220"/>
<img src="/assets/images/randomart/25.png" alt="" width="220"/>
<img src="/assets/images/randomart/26.png" alt="" width="220"/>
<img src="/assets/images/randomart/27.png" alt="" width="220"/>
<img src="/assets/images/randomart/28.png" alt="" width="220"/>
<img src="/assets/images/randomart/29.png" alt="" width="220"/>
<img src="/assets/images/randomart/30.png" alt="" width="220"/>
<img src="/assets/images/randomart/31.png" alt="" width="220"/>
<img src="/assets/images/randomart/32.png" alt="" width="220"/>
<img src="/assets/images/randomart/33.png" alt="" width="220"/>
<img src="/assets/images/randomart/34.png" alt="" width="220"/>
<img src="/assets/images/randomart/35.png" alt="" width="220"/>
<img src="/assets/images/randomart/36.png" alt="" width="220"/>
<img src="/assets/images/randomart/37.png" alt="" width="220"/>
<img src="/assets/images/randomart/38.png" alt="" width="220"/>
<img src="/assets/images/randomart/39.png" alt="" width="220"/>
<img src="/assets/images/randomart/40.png" alt="" width="220"/>
<img src="/assets/images/randomart/41.png" alt="" width="220"/>
<img src="/assets/images/randomart/42.png" alt="" width="220"/>
<img src="/assets/images/randomart/43.png" alt="" width="220"/>
<img src="/assets/images/randomart/44.png" alt="" width="220"/>
<img src="/assets/images/randomart/45.png" alt="" width="220"/>
<img src="/assets/images/randomart/46.png" alt="" width="220"/>
<img src="/assets/images/randomart/47.png" alt="" width="220"/>
<img src="/assets/images/randomart/48.png" alt="" width="220"/>
<img src="/assets/images/randomart/49.png" alt="" width="220"/>
<img src="/assets/images/randomart/50.png" alt="" width="220"/>
<img src="/assets/images/randomart/51.png" alt="" width="220"/>
<img src="/assets/images/randomart/52.png" alt="" width="220"/>
<img src="/assets/images/randomart/53.png" alt="" width="220"/>
<img src="/assets/images/randomart/54.png" alt="" width="220"/>
<img src="/assets/images/randomart/55.png" alt="" width="220"/>
<img src="/assets/images/randomart/56.png" alt="" width="220"/>
<img src="/assets/images/randomart/57.png" alt="" width="220"/>
<img src="/assets/images/randomart/58.png" alt="" width="220"/>
<img src="/assets/images/randomart/59.png" alt="" width="220"/>
<img src="/assets/images/randomart/60.png" alt="" width="220"/>

<hr>

## Appendix {#appendix}

### Usage
You can generate an image by cloning the randomart program and running it:
```
git clone https://github.com/jamesma100/randomart && cd randomart
cabal run -- randomart [--d <depth>] [--p <pixels>] [-o <filepath>] [-s <seed>]
```

For example:
```
cabal run -- randomart --d 25 --p 200 -o ./my_img.png -s my_happy_seed
```

The above images were generated in a loop with a different seed each time:
```
for i in {1..60}; do cabal run randomart -- -p 200 -o ./images/${i}.png -d 35 \
-s $(echo $RANDOM | shasum -a 256 | awk '{print $1}' /dev/stdin); done
```

### Details

The grammar I used looks like:

```haskell
ruleA = RuleNode [RandNode, RandNode, XNode, YNode]
ruleC = RuleNode
  [ruleA
  ,ruleA
  ,SinNode (AddNode ruleC ruleC)
  ,CosNode (AddNode ruleC ruleC)
  ,TanNode (AddNode ruleC ruleC)
  ,SinNode (MultNode ruleC ruleC)
  ,CosNode (MultNode ruleC ruleC)
  ,TanNode (MultNode ruleC ruleC)
 ]
ruleE = RuleNode [TripleNode ruleC ruleC ruleC]
grammar = [ruleA, ruleC, ruleE] :: Grammar
```

You can create your own grammar by composing different rules, and that'll change how your AST is constructed, and hence how your image is generated.
Rules can be anything from trigonometric functions to if-else statements. I defined these as different constructors for the same data type.

```haskell
data Node =
  NumberNode Double |
  BoolNode Bool |
  RandNode |
  SinNode Node |
  CosNode Node |
  TanNode Node |
  XNode |
  YNode |
  AddNode Node Node |
  MultNode Node Node |
  ModNode Node Node |
  ExpNode Double Node |
  TripleNode Node Node Node |
  GTNode Node Node |
  GTENode Node Node |
  LTNode Node Node |
  LTENode Node Node |
  IfNode Node Node Node |
  NormNode Node |
  NullNode |
  RuleNode [Node]
  deriving Show
```


### Hints
It's hard to predict what your generated image will look like, but in general a larger depth and a non-linear ruleset will give you more detailed images.
However, sometimes your image will still be very simple if a terminal node is selected early on and the algorithm halts.
Note that the first rule of your grammar (e.g. `ruleA` above) must be terminal, otherwise the algorithm will not halt.

For more details, you should read the [paper](https://users.ece.cmu.edu/~adrian/projects/validation/validation.pdf).
