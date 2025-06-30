---
layout: post
title: "Abstracting over Futures and Options"
---

In Scala, two interesting concepts you'll often work with are Futures and Options.
A Future is an abstraction for some value that might not be available yet.
An Option abstracts over the possibility of a value.

Both Futures and Options are monads, meaning you can chain them together without explicitly unwrapping them.
More formally, every monad `F[A]` has the following three operations [[^1]]:
- `map[B](f: A => B): F[B]`
- `flatMap[B](f: A => F[B]): F[B]`
- `pure(x: A) => F[A]`

With these, you can easily combine Futures and Options.
As a dummy example:
```scala
Future("hello").map(_ + " world").flatMap(x => Future(Some(x)))
// returns Future(Success(Some(hello world)))
```

However, what if you had a Future of an Option, i.e. `Future[Option[A]]`?
Perhaps a potentially long-running database fetch.

```scala
def getUserIdOptF: Future[Option[String]] = {
    // ...some potentially long running database fetch...
}
def getSSNOptF(s: String): Future[Option[String]] = {
    // ...get ssn if it exists...
}
```

Your code would unfortunately become more verbose:
```scala
getUserIdOptF.flatMap(_.map(num => getSSNOptF(num)).getOrElse(Future(None)))
```

Explanation: the `map()` call returns a `Option[Future[Option[A]]]` so we need the extra `getOrElse()` call so that the function passed to `flatMap()` obeys its signature, i.e. it returns a `Future[Option[A]]`.
This seems rather cumbersome, especially if many your methods share the same pattern of returning a `Future[Option[A]]`.

To reduce the boilerplate, we can instead make `Future[Option[A]]` itself a monad, so that we can simply call `flatMap()` and pass in a function with type `A => Future[Option[B]]`.

```scala
class FutureOption[A](futureOpt: Future[Option[A]]) {
  def map[B](f: A => B): Future[Option[B]] = futureOpt.map(_.map(f))
  def flatMap[B](f: A => Future[Option[B]]): Future[Option[B]] = {
    futureOpt.flatMap(_.map(f).getOrElse(Future(None)))
  }
  def pure(a: A) = Future(Some(a))
}
```

Now this works:
```scala
val userIdOptF = FutureOption[String](getUserIdOptF)
userIdOptF.flatMap(getSSNOptF)
```

We can generalize this idea even further into something that can take any monadic type and wrap it around an Option type.
This will allow us to define `List[Option[A]]` for the `List` type, `Either[Option[A]]` for the `Either` type, and similarly for every other monad that exists!

This type of thing is called a [monad transformer](https://en.wikipedia.org/wiki/Monad_transformer), which basically lets you stick a monad inside another monad.
We'll use the `OptionT` type in the [cats](https://typelevel.org/cats/datatypes/optiont.html) package, which is a handy implementation of this concept.
The same idea above can now be written much more succinctly:

```scala
import cats.data.OptionT
import cats.syntax.all._

def wrappedGetUserIdOptF: OptionT[Future, String] = OptionT(getUserIdOptF)
def wrappedGetSSNOptF(s: String): OptionT[Future, String] = OptionT(getSSNOptF(s))
wrappedGetUserIdOptF.flatMap(wrappedGetSSNOptF)
```
Note that while the outer monad can be generic, the inner is fixed to an Option.
As far as I know, there's [no easy way](https://www.reddit.com/r/haskell/comments/111y0vy/monads_doesnt_compose_well_why/) to take in two arbitrary monads `A` and `B` and compose another monad `A[B]`.
The reasoning for this is beyond my understanding.

Lastly, you can imagine dealing with functions that return Futures and Options as well.
```scala
def getSSNOfSpouseOpt(s: String): Option[String] = {
    // ...given someone's ssn, get their spouse's ssn...
}
def scrambleDigitsF(s: String): Future[String] {
    // ... scramble digits of a ssn...
}
```
[[^2]]

It would be nice to be able to combine those using the same general pattern.
Fortunately, cats provides two functions that allow you to "lift" both Options and some type `F` (in our case a Future) into an `OptionT`.

- `liftF()` lifts a function returning type `F[A]` into an `OptionT[F, A]`
- `fromOption[F]()` does the same to functions returning Options

```scala
def wrappedGetSSNOfSpouseOpt(s: String) =
    OptionT.fromOption[Future](getSSNOfSpouseOpt(s))
def wrappedScrambleDigitsF(s: String) = OptionT.liftF(scrambleDigitsF(s))
```

And we're back to where we started - one operation to rule them all:
```scala
wrappedGetUserIdOptF
    .flatMap(wrappedGetSSNOptF)
    .flatMap(wrappedGetSSNOfSpouseOpt)
    .flatMap(wrappedScrambleDigitsF)
```

---
[^1]: `map()` actually comes from [applicatives](https://en.wikipedia.org/wiki/Applicative_functor#Definition), but every monad is also an applicative so must also define it.
[^2]: You might think these examples are ridiculous, but I doubt they're beyond the abilities of a 21st Century Corporation.

