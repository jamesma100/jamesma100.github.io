---
layout: post
title: "Tail recursions"
---

I've been writing a bit of Scala lately. In particular, challenging myself to using purely functional features - no mutable variables, no shared state, no loops. It's actually easy writing recursive functions in this manner. No need to worry about dangling references, locking global data structures, or things mutating underneath you in unexpected ways. And no more defensive mechanisms to get around those issues, such as deep copying a referenced object simply because you're not sure if something else is using it. But more importantly, the Scala compiler does a great job at recycling stack frames for recursion to be inexpensive.

For instance, take this method that reverses a list:

```
// Simple recursive
def reverse1[A](li: List[A]): List[A] = {
  li match {
    case Nil => Nil
    case x :: Nil => List(x)
    case x :: rest => reverse1(rest) :+ x
  }
}
```

Since cycling through a list like this produces a stack frame allocation per recursive call, we get a nasty O(n^2) memory usage. But only because we are preserving some current state, i.e. the very last `:+ x`. If we somehow get rid of that, the compiler would no longer need the current frame, and can thus recycle it to be used by the next recursive call. This is known as [tail recursion](https://en.wikipedia.org/wiki/Tail_call), and it can be used if you want recursion but with low memory overhead. We can factor the preceding method to the following:

```
// Tail recursive
def reverse2[A](li: List[A]): List[A] = {
  def reverseTail(li: List[A], solution: List[A]): List[A] = {
    li match {
      case Nil => solution
      case x :: rest => reverseTail(rest, x :: solution)
    }
  }
  reverseTail(li, Nil)
}
```

Notice that there is no longer any remaining state in each frame when the method returns. The return value is simply a function call. In other words, instead of keeping track of the current solution in what is effectively a local variable, we are sticking it inside the method itself as an argument, in this case the `solution` variable. So every recursive call passes the current solution to the next, so on and so forth, until the end of the list is reached.

Doing some simple benchmarking, we see that with a list of 1000 elements, `reverse1()` uses about 39MB of memory. With 1500 elements, it uses 55MB. And with a list of 2000 elements, it errors out with a stack overflow. On the other hand, `reverse2()` uses just 29MB for all three cases. It can even reverse 5000 elements, or 10,000 elements. The key here is that the bottleneck is no longer memory, but the total runtime of the program.

As shown, a simple refactoring of the original solution to use tail recursion improves memory usage enormously, allowing us to write recursive functions in a way that would otherwise be intractable.

P.S. the proper way to do this functionally is not actually with recursion, but with a `foldLeft` or `foldRight`. Namely:

```
// Fold
def reverse3[A](li: List[A]): List[A] = {
  li.foldRight(Nil) {
    (a: A, b: List[A]) => {
      b :+ a
    }
  }
}
```
