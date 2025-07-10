---
layout: post
title: "Parser combinators for postal addresses"
---

A while ago, Tsoding did a [stream](https://www.youtube.com/watch?v=N9RUqGYuGfw&t=5936s&ab_channel=Tsoding) on parsing JSON with parser combinators.
I was curious how hard it would be to use the basic parsers he implemented to create a new parser for U.S. postal addresses.
It turns out once you have basic building blocks, creating new parsers is not only easy, but fun!
You can find the full code [here](https://github.com/jamesma100/postal-address-parser).

I recommend watching the video first, but if you want a dumbed down introduction to parser combinators, read on.

## What's a postal address?
To keep things simple, we will be working with a simplified version of this Wikipedia [BNF example](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form#Example).
```
<postal-address> ::= <name-part> <street-address> <zip-part>

    <name-part> ::= <first-name> <last-name> <opt-suffix-part> <EOL> |

        <opt-suffix-part> ::= "Sr." | "Jr." | ""

    <street-address> ::= <house-num> <street-name> <apt-num> <EOL>

    <zip-part> ::= <town-name> "," <state-code> <ZIP-code> <EOL>
```
An address consists of three parts: a name part, a street address, and a zip part.
Each part can take on multiple forms; for example, a name part can optionally have a suffix at the end.
We would like to parse this into some a tree, where each node represents each component.

For this simple example, using parser combinators is probably an overkill, as you can probably just brute-force parse it in Python.
However, as the postal address format changes and grows, we're going to want a more scalable approach that can adapt to these changes.
And as you will see by the end, parser combinators provide a clean abstraction that lets you easily build new parsers on top of existing ones.
You'll also realize that functional programming can actually be useful and isn't just a pretentious cult!

## Parsing characters and strings
The core idea of combinator parsing is to 1. define basic parsers as our building blocks and 2. define the _operations_ on those parsers such that they can combine with each other in various ways.

First, let's translate the BNF from the previous section to code.
```haskell
-- final address
data PostalAddress =
  PostalAddress NamePart StreetAddress ZipPart deriving (Show, Eq)
-- name part
data NamePart =
  NamePartBasic FirstName LastName |
  NamePartWithSuffix FirstName LastName SuffixPart deriving (Show, Eq)
data FirstName =
  FirstName String deriving (Show, Eq)
data LastName =
  LastName String deriving (Show, Eq)
data SuffixPart =
  SuffixPart String deriving (Show, Eq)
-- address part
data StreetAddress =
  StreetAddress HouseNum StreetName AptNum deriving (Show, Eq)
data HouseNum =
  HouseNum String deriving (Show, Eq)
data StreetName =
  StreetName String deriving (Show, Eq)
-- zip part
data ZipPart =
  ZipPart TownName StateCode ZipCode deriving (Show, Eq)
data TownName =
  TownName String deriving (Show, Eq)
data StateCode =
  StateCode String deriving (Show, Eq)
data ZipCode =
  ZipCode String deriving (Show, Eq)
data AptNum =
  AptNum String deriving (Show, Eq)
```

We can then define our parser.
```haskell
data Parser a = Parser {
  runParser :: String -> Maybe (a, String)
}
```
which is a datatype with a single field `runParser`.
You can see that it takes a string to parse, and returns a `Maybe` of something of type `a` that it parsed, along with the remainder of the string that it didn't parse.
The remaining sring can be fed as input to other parsers to continue parsing.

A couple notes on this weird syntax.
Haskell actually creates a _method_ named `runParser` that has signature
```
ghci> :t runParser
runParser :: Parser a -> String -> Maybe (a, String)
```
So you can supply the parser to it and it'll return the function field, much like accessing a field name of a struct in other languages.
This is why we named it `runParser` rather than something like `parsingFunction`, since `runParser` gives us the function of type `String -> Maybe(a, String)`, which we can run directly by providing the string argument! 

To make this more concrete, let's create our first parser for the `Char` type.
To do this, we'll first want a function that take _any_ character `c` and generate a parser for `c`.
```haskell
charParserG :: Char -> Parser Char
charParserG c = Parser f
  where
    f (x:xs)
      | x == c = Just (x, xs)
      | otherwise = Nothing
    f [] = Nothing
```

We can use this to create a parser for the letter `'h'` for instance, and use it to parse `"hello"`.

```
ghci> runParser (charParserG 'h') "hello"
Just ('h',"ello")
```
As expected, it parsed the letter `"h"` and left the rest of the string untouched.

How would we parse a string?
We could do something similar to `charParserG`, but instead of looking at a single character, we try to match an entire string.

```haskell
substr :: Int -> Int -> String -> String
substr start end s = take (end - start) (drop start s)

stringParserGDumb :: String -> Parser String
stringParserGDumb s = Parser f
  where
    f (x)
      | isPrefixOf s x = Just(x, substr (length s) (length x) x)
      | otherwise = Nothing
```
This works fine, but there is a lot of boilerplate code.
What if I told you we could _combine_ many `charParserG`'s to create a `stringParserG`?

After all, a string is simply a list of characters.
As a first attempt, we can try to call the `map` function, but unfortunately the result is of the wrong type.

```
ghci> :t map charParserG "hello"
map charParserG "hello" :: [Parser Char]
```
That is, it gives us a list of char parsers, but what we want is a parser of a list of chars.
So we need to somehow flip the parser inside out.
Luckily, there is a function that does exactly this.

```
ghci> :info sequenceA
type Traversable :: (* -> *) -> Constraint
class (Functor t, Foldable t) => Traversable t where
  ...
  sequenceA :: Applicative f => t (f a) -> f (t a)
  ...
  	-- Defined in ‘Data.Traversable’
```
`sequenceA` however, is defined in the `Functor` interface which mean we need to implement that for our `Parser` type before we can use it.

## Functor
To implement a functor, we need to define the operator `fmap`, or `<$>`, which has the following signature.
```haskell
fmap :: (a -> b) -> f a -> f b
```
It takes two arguments: a function mapping `a -> b` and a value `f a`, which is something of type `a` wrapped in some container `f`.
It then _unwraps_ the `f a` value, runs the function on the `a` to get a `b`, then _rewraps_ it, finally returning `f b`.

This is much easier to understand through an example.
Let's say we have a `Maybe` of an `Int`, and we want to double the value inside it and return another `Maybe` type.
Since the `Maybe` type is a functor, we can simply call the `fmap` operator.
```
ghci> (\i -> i * 2) <$> (Just 2)
Just 4
```
Much better than manually unwrapping the `Maybe` and doing the repackaging ourselves!
This is a trivial example but you can imagine much more complicated versions of `f` with a lot of hidden state that would be impractical to handle manually.

So for our parser, given a function and `parserA`, we need to apply the function to the output of `parserA` and return a new parser `parserB`.
```haskell
instance Functor Parser where
  -- fmap :: (a -> b) -> Parser a -> Parser b
  fmap f parserA =
    Parser $ \s ->
      case runParser parserA s of
        Nothing -> Nothing
        Just (a, as) -> Just (f a, as)
```

As an example, we can easily map the function `toUpper` to our parser for `"h"`.
```
ghci> runParser (toUpper <$> (charParserG 'h')) "hello"
Just ('H',"ello")
```

Now that `Parser` is a functor, implementing `stringParserG` is a two-line endeavor:
```
stringParserG :: String -> Parser String
stringParserG = sequenceA . map charParserG
```

It works!
```
ghci> runParser (stringParserG "hello") "hello world!!"
Just ("hello"," world!!")
```

If we go back to our postal address discussion and try to parse something simple like a `FirstName`,
we run into an issue: that is, our `stringParserG` function can only generate parsers for _specific_ strings.
But a first name can be _any_ string given some condition, such as that containing only letters in the alphabet.
This means we need another function that can generate a string parser _for some condition_.

Luckily, there is a function called `span` defined in `GHC.List` that does just this!
It takes a string and a boolean function and returns a tuple of two elements: everything it could greedily match and the rest of the string.

```
ghci> span isAlpha "hello world 123"
("hello"," world 123")
```
We can now create a function that takes in a condition and returns a parser that parses not specific strings, but _any_ string that satisfies that condition.
```haskell
spanParserG :: (Char -> Bool) -> Parser String
spanParserG f = Parser $ \s ->
  let (token, rest) = span f s
    in Just(token, rest)
```

Now we can write a parser that parses string literals by supplying it the `isAlpha` function:
```haskell
stringLiteralParser :: Parser String
stringLiteralParser = spanParserG isAlpha
```

And one that parses digits using the `isDigit` function:
```haskell
numberParser :: Parser String
numberParser = spanParserG isDigit
```

Using the `fmap` operator we defined earlier for our functor, we can now create a parser for `FirstName`.
```haskell
firstNameParser :: Parser FirstName
firstNameParser = FirstName <$> stringLiteralParser
```

Doing the same for `LastName`:
```haskell
lastNameParser :: Parser LastName
lastNameParser = LastName <$> stringLiteralParser
```

We can try to write a parser for `SuffixPart` but there is another problem.
Recall that a suffix can be _either_ "Sr." or "Jr." so we'll need a way to apply the "or" operator on two parsers, such that we return the first match that succeeds.
Then we can write a parser for "Sr." and a parser for "Jr." and then glue them together.

This can be done with the `<|>` operator, defined in yet another interface, the `Alternative`.
But to implement `Alternative`, we must first implement the `Applicative` interface.
So once again, we're going to take a detour into some PL theory.

## Applicative
An applicative has these two operations:
```
pure :: a -> f a 
(<*>) :: f (a -> b) -> f a -> f b
```
`pure` simply wraps some primitive type `a` inside our container `f`.
The `<*>` operator looks similar to `<$>` in its type signature, but notice that in this case, the function is also wrapped inside the container!

We use it like this:
```
ghci> (Just toUpper) <*> (Just 'h')
Just 'H'
```

The applicative implementation is similar to what we did for our functor, except we also need to "unwrap" the container wrapping the function.
In this context, unwrapping our container means running the parser!
```haskell
instance Applicative Parser where
  -- pure :: a -> Parser a
  pure a = Parser $ \s -> Just (a, s)

  -- <*> :: Parser (a -> b) -> Parser a -> Parser b
  parserF <*> parserA = Parser $ \s ->
    case runParser parserF s of
      Nothing -> Nothing
      Just (f, fs) ->
        case runParser parserA fs of
          Nothing -> Nothing
          Just (a, as) -> Just (f a, as)
```

## Alternative
Now that our parser is an applicative, we can continue on to implement `Alternative`.
In this case, we need the `empty` operator and the `<|>` operator, which we will use as a "or" function combining our parsers.

To implement the `<|>` operator, we need to parse our string with the first parser, and depending on the results, potentially run it through the second parser.

```haskell
instance Alternative Parser where
  -- empty :: f a
  empty = Parser (\s -> Nothing)

  -- (<|>) :: f a -> f a -> f a
  parserA <|> parserB = Parser $ \s ->
    case runParser parserA s of
      Just (a, as) -> Just (a, as)
      Nothing -> case runParser parserB s of
        Just (b, bs) -> Just (b, bs)
        Nothing -> Nothing
```

Now that our `Parser` is both an `Applicative` and an `Alternative`, we can write our suffix parser.

```haskell
suffixPartParser :: Parser SuffixPart
suffixPartParser = SuffixPart <$> ((stringParserG "Jr.") <|> (stringParserG "Sr."))
```

Remember that a `NamePart` can optionally have a suffix.
So our final parser has to look something like this:
```haskell
namePartParser :: Parser NamePart
namePartParser = namePartWithSuffixParser <|> namePartBasicParser
```
which tries to match the name part with a suffix, but if it fails falls back to a basic name.

How do we implement `namePartWithSuffixParser` and `namePartBasicParser`?
For these, we now need to run a _sequence_ of parsers on some input string, each parser parsing the remainder of the input string left behind by its preceding parser.
To do this, we need the "bind" operator, or `>>=` defined in the `Monad` interface.

```
ghci> :info >>=
type Monad :: (* -> *) -> Constraint
class Applicative m => Monad m where
  (>>=) :: m a -> (a -> m b) -> m b
  ...
  	-- Defined in ‘GHC.Base’
infixl 1 >>=
```
So before we can finish our parser, we need to do one last task, which is to implement the `Monad` interface.
But once we have this, the rest will be smooth sailing.

## Monad
Simply enough, to implement `>>=`, we just run the first parser on the input, take the `Maybe` result and run the second parser on the rest of the input.
```haskell
instance Monad Parser where
  -- >>= :: (Parser a) -> (a -> Parser b) -> (Parser b)
  parser >>= f = Parser $ \s ->
    case runParser parser s of
      Nothing -> Nothing
      Just (a, as) -> runParser (f a) as
```

Consider the follwing parser that parses `"good"`.
```
ghci> runParser (stringParserG "good") "goodbye"
Just ("good","bye")
```
If we want to do the same thing but additionally thread the remaining string, i.e. `"bye"`, into another parser that consumes `"bye"`, we can!
```
ghci> runParser ((stringParserG "good") >>= (\s -> stringParserG "bye")) "goodbye"
Just ("bye","")
```

## Impelmenting our parser
The hard part is done!
Now we can implement a parser for basic names and a parser for names with suffixes.
To do this, we can make use of the `do` notation for monads and call the smaller parsers we implemented earlier, i.e. `firstNameParser`, `lastNameParser`, and `suffixPartParser`.
Then finally, we can `<|>` the two parsers together.

```haskell
namePartBasicParser :: Parser NamePart
namePartBasicParser = do
  fst <- firstNameParser <* charParserG ' '
  snd <- lastNameParser
  pure $ NamePartBasic fst snd

namePartWithSuffixParser :: Parser NamePart
namePartWithSuffixParser = do
  fst <- firstNameParser <* charParserG ' '
  snd <- lastNameParser <* charParserG ' '
  thrd <- suffixPartParser
  pure $ NamePartWithSuffix fst snd thrd

namePartParser :: Parser NamePart
namePartParser = namePartWithSuffixParser <|> namePartBasicParser 
```

Note the `<*` operator defined in the `Applicative` interface.
It sequentially applies two parsers, but only returns the result of the first.
As an example, if we want to parse `"good night"` followed by an exclamation point, but only want the string `"good night"`, we can do something like:
```haskell
ghci> runParser ( stringParserG "good night" <* charParserG '!') "good night!"
Just ("good night","")
```

Next, parsing the address part of the postal address should look familiar.

```haskell
houseNumParser :: Parser HouseNum
houseNumParser = HouseNum <$> numberParser

streetNameParser :: Parser StreetName
streetNameParser = StreetName <$> (stringLiteralParser <* stringParserG " St")

aptNumParser :: Parser AptNum
aptNumParser = AptNum <$> ((stringParserG "Apt ") *> numberParser)

streetAddressParser :: Parser StreetAddress
streetAddressParser = do
  houseNum <- houseNumParser <* charParserG ' '
  streetName <- streetNameParser <* charParserG ' '
  aptNum <- aptNumParser
  pure $ StreetAddress houseNum streetName aptNum
```

Then lastly, the zip portion of the address:
```haskell
townNameParser :: Parser TownName
townNameParser = TownName <$> stringLiteralParser

-- only the West Coast exists
stateCodeParser :: Parser StateCode
stateCodeParser = StateCode <$>
  (stringParserG "WA" <|>
    stringParserG "CA" <|>
    stringParserG "OR" <|>
    stringParserG "AR" <|>
    stringParserG "HI")

zipCodeParser :: Parser ZipCode
zipCodeParser = ZipCode <$> numberParser

zipPartParser = do
  town <- townNameParser <* charParserG ','
  state <- stateCodeParser <* charParserG ' '
  zip <- zipCodeParser
  pure $ ZipPart town state zip
```

The final step is to create postal address parser by combining the name, address, and zip parsers!
```haskell
postalAddressParser :: Parser PostalAddress
postalAddressParser = do
  namePart <- namePartParser <* charParserG '\n'
  streetAddress <- streetAddressParser <* charParserG '\n'
  zipPart <- zipPartParser
  pure $ PostalAddress namePart streetAddress zipPart
```

We can test this on a couple examples:
```haskell
ghci> -- test case 1
ghci> runParser postalAddressParser "Linus Torvalds Jr.\n1538 John St Apt 550\nSeattle,WA 98702"
Just (PostalAddress (NamePartWithSuffix (FirstName "Linus") (LastName "Torvalds") (SuffixPart "Jr.")) (StreetAddress (HouseNum "1538") (StreetName "John") (AptNum "550")) (ZipPart (TownName "Seattle") (StateCode "WA") (ZipCode "98702")),"")
ghci> -- test case 2
ghci> runParser postalAddressParser "Grace Hopper\n528 Madison St Apt 702\nFremont,CA 78771"
Just (PostalAddress (NamePartBasic (FirstName "Grace") (LastName "Hopper")) (StreetAddress (HouseNum "528") (StreetName "Madison") (AptNum "702")) (ZipPart (TownName "Fremont") (StateCode "CA") (ZipCode "78771")),"")
```

These are admittedly pretty simple examples but you can easily build on this to create more complicated parsers to deal with your favorite edge cases.

## Summary
To recap, parser combinators are powerful abstractions that allow you to combine small, simple parsers in different ways in order to build larger, more complicated parsers.
We started with a simple parser that can parse a single character, then built more parsers on top of that to parse strings, numbers, names, street names, zip codes, and finally, entire postal addresses.
To do this, we implemented various interfaces for our `Parser` type, including `Functor`, `Applicative`, `Alternative`, and `Monad`, each of which gave us some unique operator we used to glue our various parsers together.

For more on parser combinators, I recommend this list of resources:
- [You could have invented Parser Combinators](https://theorangeduck.com/page/you-could-have-invented-parser-combinators)
- [Tsoding: JSON parsing in Haskell](https://www.youtube.com/watch?v=N9RUqGYuGfw&t=4600s&ab_channel=Tsoding)
- [Tsoding: parser combinator library in OCaml](https://www.youtube.com/watch?v=Y5IIXUBXvLs)
- [What's in a parser combinator?](https://remusao.github.io/posts/whats-in-a-parser-combinator.html)

And if you really must, the whitepaper:
- [Monadic Parser Combinators](https://people.cs.nott.ac.uk/pszgmh/monparsing.pdf)
