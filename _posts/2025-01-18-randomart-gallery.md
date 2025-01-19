---
layout: post
title: "RandomArt image gallery"
---

In a [paper](https://users.ece.cmu.edu/~adrian/projects/validation/validation.pdf) from CMU, security researchers describe a nifty algorithm used to generate random images known as _RandomArt_.
In essence, it takes as input a seed to a pseudo-random number generator and uses it to construct an [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) by selecting rules from a predefined grammar.
The AST describes a function mapping (x, y) pixel values to RGB values.

Recently I implemented a [version of it](https://github.com/jamesma100/randomart) in Haskell, and this page shows some images I was able to generate while experimenting with different grammars, which I've included at the end. Each column uses a different initial seed, so you can see that using a different seed results in a wildly different image, making it suitable for quickly spotting changes in, for instance, SSH public keys.

The outputs tend to be highly non-deterministic, but with a complicated enough grammar and deep enough tree, you can be certain the resulting image will be fascinating.

<br>
<hr>

_grammar 0, depth=8_

<img src="/assets/images/randomart/0-0-8.png" alt="" width="220"/>
<img src="/assets/images/randomart/0-1-8.png" alt="" width="220"/>
<img src="/assets/images/randomart/0-2-8.png" alt="" width="220"/>

_grammar 0, depth=12_

<img src="/assets/images/randomart/0-0-12.png" alt="" width="220"/>
<img src="/assets/images/randomart/0-1-12.png" alt="" width="220"/>
<img src="/assets/images/randomart/0-2-12.png" alt="" width="220"/>

_grammar 0, depth=20_

<img src="/assets/images/randomart/0-0-20.png" alt="" width="220"/>
<img src="/assets/images/randomart/0-1-20.png" alt="" width="220"/>
<img src="/assets/images/randomart/0-2-20.png" alt="" width="220"/>

_grammar 0, depth=24_

<img src="/assets/images/randomart/0-0-24.png" alt="" width="220"/>
<img src="/assets/images/randomart/0-1-24.png" alt="" width="220"/>
<img src="/assets/images/randomart/0-2-24.png" alt="" width="220"/>

_grammar 1, depth=8_

<img src="/assets/images/randomart/1-0-8.png" alt="" width="220"/>
<img src="/assets/images/randomart/1-1-8.png" alt="" width="220"/>
<img src="/assets/images/randomart/1-2-8.png" alt="" width="220"/>

_grammar 1, depth=12_

<img src="/assets/images/randomart/1-0-12.png" alt="" width="220"/>
<img src="/assets/images/randomart/1-1-12.png" alt="" width="220"/>
<img src="/assets/images/randomart/1-2-12.png" alt="" width="220"/>

_grammar 1, depth=20_

<img src="/assets/images/randomart/1-0-20.png" alt="" width="220"/>
<img src="/assets/images/randomart/1-1-20.png" alt="" width="220"/>
<img src="/assets/images/randomart/1-2-20.png" alt="" width="220"/>

_grammar 1, depth=24_

<img src="/assets/images/randomart/1-0-24.png" alt="" width="220"/>
<img src="/assets/images/randomart/1-1-24.png" alt="" width="220"/>
<img src="/assets/images/randomart/1-2-24.png" alt="" width="220"/>

_grammar 2, depth=8_

<img src="/assets/images/randomart/2-0-8.png" alt="" width="220"/>
<img src="/assets/images/randomart/2-1-8.png" alt="" width="220"/>
<img src="/assets/images/randomart/2-2-8.png" alt="" width="220"/>

_grammar 2, depth=12_

<img src="/assets/images/randomart/2-0-12.png" alt="" width="220"/>
<img src="/assets/images/randomart/2-1-12.png" alt="" width="220"/>
<img src="/assets/images/randomart/2-2-12.png" alt="" width="220"/>

_grammar 2, depth=20_

<img src="/assets/images/randomart/2-0-20.png" alt="" width="220"/>
<img src="/assets/images/randomart/2-1-20.png" alt="" width="220"/>
<img src="/assets/images/randomart/2-2-20.png" alt="" width="220"/>

_grammar 2, depth=24_

<img src="/assets/images/randomart/2-0-24.png" alt="" width="220"/>
<img src="/assets/images/randomart/2-1-24.png" alt="" width="220"/>
<img src="/assets/images/randomart/2-2-24.png" alt="" width="220"/>

<hr>

<h2> Appendix </h2>
The three grammars used are defined below.
Note that I didn't implement probabilities like described in the paper, so the the algorithm randomly selects a node from each rule with equal likelihood.
You can adjust this by including duplicate nodes to increase its probability. For example, Grammar 2 is the same as Grammar 0 but with adjusted probabilities of 1/4, 3/8, and 3/8.
```haskell
-- Grammar 0
let ruleA = RuleNode [RandNode, XNode, YNode]
let ruleC = RuleNode [ruleA, AddNode ruleC ruleC, MultNode ruleC ruleC]
let ruleE = RuleNode [TripleNode ruleC ruleC ruleC]
let grammar = [ruleA, ruleC, ruleE] :: Grammar
```

```haskell
-- Grammar 1
let ruleA = RuleNode [RandNode, RandNode, XNode, YNode]
let ruleC = RuleNode [ruleA,
                      ruleA,
                      SinNode (AddNode ruleC ruleC),
                      CosNode (AddNode ruleC ruleC),
                      TanNode (AddNode ruleC ruleC),
                      SinNode (MultNode ruleC ruleC),
                      CosNode (MultNode ruleC ruleC),
                      TanNode (MultNode ruleC ruleC)]
let ruleE = RuleNode [TripleNode ruleC ruleC ruleC]
let grammar = [ruleA, ruleC, ruleE] :: Grammar
```

```haskell
-- Grammar 2
let ruleA = RuleNode [RandNode, XNode, YNode]
let ruleC = RuleNode [ruleA,
                      ruleA,
                      AddNode ruleC ruleC,
                      AddNode ruleC ruleC,
                      AddNode ruleC ruleC,
                      MultNode ruleC ruleC,
                      MultNode ruleC ruleC,
                      MultNode ruleC ruleC]
let ruleE = RuleNode [TripleNode ruleC ruleC ruleC]
let grammar = [ruleA, ruleC, ruleE] :: Grammar
```
