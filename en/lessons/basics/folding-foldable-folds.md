---
version: 1.0.0
title: Folding, Foldable and Folds
---

## Introduction

_Folding_ is a form of computation that aggregates many values into a single whole by successive applications of a single step. 

The beauty of folding shines from the fact that can be used to express almost any form computation: any combination of _map_, _filter_, _reduce_ is a variety of folding; and any recursion can be done with folding. It is thus ubiquitous in programming languages and a cornerstone to Haskell. 

This lesson is divided into two parts:

1. In the first we shall take a look at folding in Haskell using built-in functions from the `Prelude` and `Control.Fold`. After a few examples we will generalize how folding works, culminating in the _Foldable_ typeclass. 

2. In the second part we will take a look at a handful of problems that folding runs into, and look at a solution offered by a third-party package (_fold_).

{% include toc.html %}

## Folding

### First example: folding into a empty container

Let's say we have a list:

```
[1,2,3,4,5] :: [Int]
```

from which we want to remove values bigger than 3. Let's say we don't know that Haskell provides the `filter` function:

```
filter :: [a] -> (a -> Bool) -> [a]
```

We can still implement such a function with folding. Here's what we need:

- a container: to hold the result of the folding (here: to hold the values below 3)
- a "computation": to operate on every value, on after the other (here: to check, for every element presented to it, whether it is bigger than three)
- a step function: to take the result of the "computation" into the container.

We can see that the "computation" and the step functions work in a single pass: in fact, the computation could also be seen as a part of the step function, i.e. the part that applies to the value, while the rest of the step function plays the role of extracting the result from the original data structure into the container. So we can simplify our above "shopping list" to:

- a container: to hold the result of the folding (here: to hold the values below 3)
- a step function: to operate on every value, on after the other, a combines the results with the container.

In Haskell, this is all we need to implement our simple filter algorithm:

```
foldl (\acc val -> if val > 3 then val : acc else acc) [] [1,2,3,4,5]
```

Let's zoom in on the parts of this expression that correspond to our shopping list:

```
    (\acc val -> ...val... : acc ) <-- step function
        [] <-- container
```

So our step function:

1. takes as input a 2-place function, that is, a function taking an `acc` and a `val`; then
2. it does something with the `val` (not showns -- the "...val..." part), and _combines_ the results with the `acc`, which here means writing the value into the container.

Here notice that the operation by means of which values are combined into the container is `cons`:

```
cons :: a -> [a] -> [a]
```

`cons` has an infix notation: (`:`). It comes in handy since, by definition, the step function in folding combines exactly two things. Infix notation captures this fact by reflecting the relational character of the combining operation.


### Second example: folding in place

Notice that, in the above example, folding was defined in terms of processing individual values from a collective data structure (`[1,2,3,4,5]`) into another collective data structure (the empty `[]`). But folding does not necessarily yield a collective data structure. This is because one can define a step function which fulfils its combination duty without the mediation of a collective datastructure to aggregate the values as they come out of the step function. It can simply combines two values into one, and use the result as the value with which the next value to be processed will combined with. Look at this:

```
foldl (\acc val -> acc + val) 0 [1,2,3,4,5]
```

The output of this function is 15. Instead of using an empty container data structure, the step function combined values -- digits -- directly, thanks to the addition operation. In other words, `acc` didn't accumulate by means of storing a sequence of values; it "accumulated" the sum of all the numbers that preceded the number in the `val` variable at any point. And taking advantage of the infix notation, we can rewrite this to:

```
foldl (+) 0 [1,2,3,4,5]
```

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
0. (+0)             -- start
1. (+0(+1))
2. (+0(+1(+2)))	0   -- finish, returning 0 now
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