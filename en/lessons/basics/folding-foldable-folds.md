---
version: 1.0.0
title: Folding, Foldable and Folds
---

## Introduction

_Folding_ is a form of computation that aggregates many values into a single whole by successive applications of a step function. 

The beauty of folding shines from the fact that can be used to express almost any form computation: any combination of _map_, _filter_, _reduce_ is a variety of folding; and any recursion can be done with folding. It is thus ubiquitous in programming languages and a cornerstone to Haskell. 

This lesson is divided into two parts:

1. In the first we shall take a look at folding in Haskell using built-in functions from the `Prelude` and `Control.Fold`. After a few examples we will generalize how folding works, culminating in the _Foldable_ typeclass. 

2. In the second part we will take a look at a handful of problems that folding runs into, and look at a solution offered by a third-party package (_fold_).

{% include toc.html %}

## Folding

### First example: folding into a container

Let's say we have a list:

```
[1,2,3,4,5] :: [Int]
```

from which we want to keep only the values above 3 and convert to strings, which finally want to concatenate. We could do this with a composition of

```
filter :: [a] -> (a -> Bool) -> [a]
map :: (a -> b) -> [a] -> [b]
concat :: [a] -> [a] -> [a]
```
namely:

```
concatMapFilterAbove3 = concat . map show . filter (> 3)
```

or more generally:

```
concatMapFilter f p elements = concat . map f . filter p $ elements
```

Defined in this way, our `concatMapFilter` function means to apply `map`, `filter` and `concat` sequentially: first by filtering every element; then by mapping them; finally by concatenating them. 

Now the Haskell compiler is clever enough to figure out that there is no gain from working sequentially here, and so kind as to transform the mapping and the filtering so that every element is filtered, mapped and concatenated in one go. But this is not cool for at least two reasons:

1. Conceptually it was never our intention to express the notion of _traversing the list several times_; in point of fact we would be happy to traverse it once.
2. We relied on the compiler to understand something different (and optimized) from what our code means.

Better would be to express what we really mean ourselves (a single traversal). And surprise! this can be done with a `foldl` as in:

```
mapFilterAbove3 =
    foldl (\acc v -> if v > 3 then acc <> show v else acc) mempty
```

or more generally:

```
mapFilter f p = 
    foldl (\acc v if p v then acc <> f v else acc) mempty
```

Now the mapping, the filtering and the concatening are internal to the `(\... -> ...)` function. The fact that the list is traversed once is made explicit as we no longer rely on `map` and `filter`. 

Good. So how does `foldl` work?

```
foldl :: Foldable t => (b -> a -> b) -> b -> t a -> b
```

This is less scary if we ignoring the outermost `->`. It boils down to

```
(b -> a -> b)   step function
b               initial value
t a             t-"foldable" container of a-values
b               final value
```

Our earlier example instantiates this as:

```
([Char] -> Int -> [Char])   step function
[Char]                      initial (empty) string
[Int]                       list of Int's
[Char]                      final string
```
Conceptually this computation involves two steps:
1. (base case)
    - the initial empty string is passed to the first argument of the step function
    - the head `Int` of the `[Int]` is passed to the second argument of the step function
    - the step function then either yields an empty string (if the value was not bigger than 3) or a modified string (with the stringified valued appended).
2. (until the container is empty) the string is appended all the other stringified `Int`s bigger than 3 as they are extracted from `[Int]`.

With this you can see already what folding is all about: 
- from
    - a container structure holding values; and
    - a step function
- folding over the container is: 
    - to consume values as they are extracted from the container one by one, combining the results into a value that is fed back to the step function until the container has been fully consumed.

### Second example: reducing

In the above example, folding was defined in terms of processing individual values from a container data structure (`[1,2,3,4,5]`) into another container data structure -- the empty `[Char]`. But folding does not necessarily yield a container data structure -- it can yield any type of value.

"Reducing" is a how colloquially one refers to a step function that combines its inputs as an output that is not a container itself. Consider:

```
foldl (\acc val -> acc + val) 0 [1,2,3,4,5]
```

more readable when rewritten to

```
foldl (+) 0 [1,2,3,4,5]
```

Here the step function takes two `Int`s to their sum; it does not accumulate by injecting one into another. So _folding into a container_ and _reducing_ are two special cases of the general form of computation, just _folding_, summarized at the end of the previous section. 

### Folding directions: foldl vs foldr

Let zoom in on again on `foldl`, in particular, on its terminal _l_. This "l" stands for "left fold". A left fold folds stuff from left-to-right. Consider again:

```
foldl (+) 0 [1,2,3,4,5]
```

If we decompose this computation into the sequence of applications of the step function, we get:

```
--  numbering applications of the step function
0. (1) (1,2,3,4,5)  -- start
1. (0+1) (2,3,4,5)
2. (1+2) (3,4,5)
3. (3+3) (4,5)
4. (6+4) (5)
5. (10+5)           -- finish
```

In point of fact there is a math word to express the notion of "left-to-right" in the folding carried out by `foldl`: _left-associative_. Here the way in which folding is left-associative can be seen more clearly if we rewrite the above as:

```
0. (0)                      -- start
1. ((0+)1)
2. (((0+)1+)2+)
3. ((((0+)1+)2+)3+)
4. (((((0+)1+)2+)3+)4+)
5. ((((((0+)1+)2+)3+)4+)5)  -- finish
```

Here you can recognize our old friends `acc`, `val` as part of the step function. At every step, `acc` takes for value the result of the left most expressions, i.e. ((0)+1) = 1 after step 1, (((0)+1)+2) = 3 after step 2, etc. Similarily, `val` takes for value the next number in the sequence of folding over [1,2,3,4,5], i.e. __1__ at step 1 and __2__ at step 2.

So the left-associativity of `foldl` is captured by the fact that the combining of values happens by a way of left-associating the `acc` and `val`. There are two important consequences of this for the way one programs in Haskell with left folds:
- the "accumulator" variable is evaluated after each application of the step function (here `acc` + `val`);
- the data structure from which the step function acquires input must be entirely consumed for evaluation to occurs.

The last observation follows from the fact that the binary operation doing the folding is _itself_ part of the thing that is evaluated with each application of the step function. The folding is not considerated complete until _all_ the values folded over have been related (here: added) to one another in succession.

In most cases, a Haskell programmer will be happy with these two aspects of `foldl`. That is, it will be desired that evaluation drives the consumption of values one is folding over. That is typically the case when trying to use the folded over values -- be it the single value obtained from a binary operation like addition, or all of them individually, or the collection consisting of all of them as a collective.

To give an example of the latter flavour, suppose we're folding over a list of words (`[Data.Text]`) trying to intercalate a space between them:

```
let t = foldl (\ws w -> ws ws `Data.Text.append` w)
            mempty ["Let","it","be!"]
in  print t 
```

Here the list of texts -- words in facts -- are folded over by appending them (in sequence) to the `mempty` value, which in the context of folding over a list of texts stands for an empty text. We are glad that the accumulator variable (`ws`) is evaluated after each append; and we are glad that the data structure from which the step function acquires input (the list of texts/words) must be entirely consumed. We want the entire text and we want it now! (We need it for printing to the screen after all.)

However there are cases we are unhappy with these two properties:
- we don't want the "accumulator" variable to be evaluated after each application of the step function (perhaps we don't need evaluation at all)
- we don't want the data structure from which the step function acquires input to be entirely consumed just so that we can use the result.

We don't want these two things, typically, when we are folding over a data structure of indeterminate (or equivalently: of infinite) length, or when we are not equally interested in all the values produced from folding. Consider an example that illustrates both requirements: summing all elements up to a certain value, extract the sum and stopping there:

```
foldr (\val acc -> if val < 3 then acc + val else 0) 0 [1..]
```

`[1..]` lists all `Int`s from 1 on. This is, of course, an infinite list. Yet it evaluates in a few milliseconds. That's because it benefits from early termination. Where does the early termination come from?

Since we're using `foldr`, the folding happens right-to-left. On its first run, the step function evaluates `0 < 3`. It checks, so it adds 1 to 0. On the second run, it checks, so it adds 2 to 1. On the third run, it does not check, so the step function returns 0. Why does passing 0 makes the function terminate? Look closer at how this works:

```
0. (0)             -- start
1. (+1(+0))
2. (+2(+1(+0)))	0   -- finish, as adding one more 0 terminated the sequence
```

After the second application of the step function, so after the second step, the function stops because passing `0` to the accumulator exhausted it. Indeed, the expression `(+0(+1(+2)))` becomes "complete" (terms of art: saturated) because passing it 0 makes it evaluate to 0+(+0(+1(+2))), which is indeed equal to 3.

So here's the important difference between `foldr` and `foldl`: a computation driven by `foldr` early terminates when and only when the step function evaluates to a value which "dries out" the accumulator, that is, right-associates with the accumulated values to yield a complete expression. Because `foldl` associates to the left, it must consume the entire input in order to yield a complete expression.

Our takeway for the practitioner thus boils down to:
- use `foldr` when you need early termination or folding over an infinitely or indeterminately long data structure;
- use `foldl` otherwise, in particular when you need to aggregate a single value from consuming an entire container of multiple values.

## Foldable and Monoids

Haskell is kind enough to offer the _Foldable_ typeclass to let the programmer call `foldl` and `foldr` on a variety of types besides lists (maps, sets, hashmaps, hashtables, trees, tries, sequences, etc.) In general a type belongs to _Foldable_ just in case it instantiates either `foldMap` or `foldr`. We have seen `foldr` in the previous section so let us take a look at `foldMap`:

```
foldMap :: Monoid m => (a -> m) -> t a -> m
-- For example:
foldMap show [1,2,3]
> "123"
```

`foldMap` works like this:
1. it maps the function passed as first argument to the elements in the data structure passed in the second argument to a result
2. it combines the results of (1) as they come to a single value.

In our example it maps the numbers to their representation as strings, and combines all 1-character strings into a single 3-character string.

If we squint a little we see it combines two familar signatures:

```
--  simplified signature
($) :: (a -> b) -> a -> b
```

which represents the computation described in (1) above, and

```
fold :: Monoid m => t m -> m
```

which describes the computation described in (2). The _Monoid_ constraint on the signature is a hint that, as far as monoids are concerned, `fold` is just a special case of `foldr`:

```
-- assuming that 'm' is a monoid
fold m = foldr (\val acc -> val <> acc) mempty m
```

So it begins to look like `foldMap` as a special case of `fold`, namely, a `foldr` operating on a monoid with `ap` sprinkled on top:

```
-- assuming that 'm' is a monoid
--  '$' is unnecessary here and used only for illustration
foldMap f m = foldr (\val acc -> (f $ val) <> acc) mempty m
```

In view of this equality, it's not a surprise that there is no need to define _both_ an instance of `foldMap` and an instance of `foldr` to get an instance of _Foldable_. Defining any one of them is enough, since `foldMap` is a special case of `foldr` applied to monoids. The relationship is even more immediate if we take lists for monoids. This can be seen from one way of defining `foldr` and `foldMap`for the list typeclass:

```
--  not the real deal, but close enough
instance Foldable [] where
    foldr step acc [] = acc
	foldr step acc (x:xs) = step x $ foldr step acc xs
```

and:

```
--  not the real deal, but close enough
instance Foldable [] where
	foldMap step [] = mempty
	foldMap step (x:xs) = step x : foldMap step xs
```

which reveals that in particular:

```
foldr (\val acc -> f val <> acc) mempty (x:xs) = foldMap f (x:xs)
```

Refer to [here](https://hackage.haskell.org/package/base-4.16.0.0/docs/src/GHC.Base.html#foldr) for the real thing.

## Folds