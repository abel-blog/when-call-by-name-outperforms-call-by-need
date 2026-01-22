When call-by-name outperforms call-by-need
==========================================

_TL;DR: In call-by-need, sharing list generators might be a source of space leaks, and you might want to explicity _unshare_ them, simulating call-by-name._

## Call-by-need takes the best of call-by-value and call-by-name, or does it?

I have been teaching evaluation stategies for many years, advertising call-by-need as the _Synthese_ of call-by-value and call-by-name:  Call-by-value (_These_) might evaluate unused function arguments, call-by-name (_Antithese_) does not do such but might evaluate arguments several times, yet call-by-need evaluates each argument at most once, and only when needed.  So it takes the best of two worlds, or does it?

I knew from programming practice that this is a gross oversimplification, and call-by-need complicates the semantics by making a heap indispensable, and there are all sorts of difficulties with space leaks through thunk retainment etc.

However, when studying space leaks in earnest for the first time, I was still surprised to come across a simple example where call-by-need is outperformed not by call-by-value but by the unlikely _call-by-name_.  In this example, the "at most once" property of argument evaluation comes not as a blessing, but as a bane.  The example is a Haskell one-liner:
```haskell
main = print $ head $ drop (10^8) $ cycle [1..10^7]
```
Here, `[1..10^7]` is syntax for the generator `enumFromTo 1 (10^7)` that lazily produces the list  `[1,2,...,10^7]`.

This program swallows huge amounts of space, it constructs the whole list in memory.
Why is that?

Recall the definition of `cycle`:
```haskell
cycle xs = xs ++ cycle xs
```

## Call-by-name evaluation

Let us first look at how `head $ drop (10^8) $ cycle [1..10^7]` would evaluate in call-by-name:
```haskell
    head $ drop (10^8)   $ cycle [1..10^7]
--> head $ drop (10^8)   $ [1..10^7]     ++ cycle [1..10^7]
--> head $ drop (10^8)   $ 1 : [2..10^7] ++ cycle [1..10^7]
--> head $ drop 99999999 $ [2..10^7]     ++ cycle [1..10^7]
--> head $ drop 99999999 $ 2 : [3..10^7] ++ cycle [1..10^7]
--> head $ drop 99999998 $ [3..10^7]     ++ cycle [1..10^7]
--> ...
--> head $ drop 90000001 $ 10^7 : []     ++ cycle [1..10^7]
--> head $ drop 90000000 $ []            ++ cycle [1..10^7]
--> head $ drop 90000000 $ cycle [1..10^7]
--> head $ drop 90000000 $ [1..10^7]     ++ cycle [1..10^7]
--> ...
--> head $ drop 80000000 $ cycle [1..10^7]
--> ...
--> head $ drop 10000000 $ cycle [1..10^7]
--> ...
--> head $ drop 0 $ cycle [1..10^7]
--> head $ cycle [1..10^7]
--> head $ [1..10^7] ++ cycle [1..10^7]
--> head $ 1 : [2..10^7] ++ cycle [1..10^7]
--> 1
```
The whole sequence runs in constant memory!
So, what goes wrong in call-by-need?

## Call-by-need evaluation

In call-by-need, evaluating `cycle [1..10^7]` allocates cell `xs = [1..10^7]` on the heap and proceeds to evaluate `xs ++ cycle xs`.  As the first element of this list is dropped, `xs` is evaluated to `xs = 1 : xs1` with new cell `xs1 = [2..10^7]`, transforming the list to `xs1 ++ cycle xs`. Dropping the next element results in `xs2 ++ cycle xs` with new cell `xs2 = [3..10^7]` and the update `xs1 = 2 : xs2`. This continues, until `xs9999999 = [10^7]`.
Since `xs` is still needed for the recursive call `cycle xs`, it cannot be freed, and consequently, none of the other 9999999 cells can.  Thus, we end up with a massive space leak of 10 million entries!

The problem here is that the generator `[1..10^7]` is shared through cell `xs` in call-by-need, while in call-by-name it was simply duplicated.  The generator itself is a very small closure and it produces the next list element each time in negligible runtime, thus there is no harm in duplicating it.  In contrast, the list _it produces_ is large and should not duplicated but shared, and this is what the call-by-need strategy aims and succeeds to achieve, yet with harmful consequences.

If we could articulate that we do not want computation to be shared on our call to `cycle` we would be well off.  Haskell does not offer us a primitive for that, but we can employ _manual thunking_, a trick used in call-by-value languages to prevent eager evaluation.

## Forcing call-by-name

We implement a version of `cycle` with an explicitly thunked argument:
```haskell
cycle :: (() -> [a]) -> [a]
cycle f = f () ++ cycle f

main = print $ head $ drop (10^8) $ cycle $ \ () -> [1..10^7]
```
This program now runs in constant space, following the trace we rolled out for call-by-name above.  Not the list `xs`, only the delayed iterator `f = \ () -> [1..10^7]` gets shared an its cell `f` is never updated because its content is already a value (a function value).

Conclusion
----------

Laziness and call-by-need promise "care-free" functional programming and indeed, from a purely logical standpoint, the techinque is persuasive.  However, once resource consumption enters the picture, the elegance of lazy languages gets jeopardized.  Simple intuitions about evaluation behavior break down, and code needs to be scrutinized for space leaks on a case-by-case basis.

Call-by-value languages do not suffer from such problems, the mental model for evaluation is comparatively simple and it is easier to intuitively understand space consumption behavior.  On the other hand, call-by-value hampers abstraction and function reuse [as Lennard Augustsson argues](https://augustss.blogspot.com/2011/05/more-points-for-lazy-evaluation-in.html).

I don't want to offer definitive conclusions, but it seems safe to say that abstraction and efficiency are still antitheses, and we need to keep searching for the Holy Grail.

On a more mundane basis, maybe it would be worthwile discussing space leak dangers more in the Haskell documentation, e.g. in the one of `cycle`.  I might also be worthwile complementing `cycle` with her call-by-name sister `cycle'`, and discuss their respective advantages.

Update
------

v0.2: [Jaro Reinders educated me](https://discourse.haskell.org/t/when-call-by-name-outperforms-call-by-need-space-leaks-with-cycle-etc/13586/6?u=andreasabel) that my trick with explicit thunking is thwarted by GHC's _full laziness_ "optimization" [`-ffull-laziness`](https://downloads.haskell.org/ghc/9.14.1/docs/users_guide/using-optimisation.html#ghc-flag-ffull-laziness) that is turned on with `-O`.
Full laziness lifts expressions from under binders when possible, turning my explicitly thunked version into:
```haskell
main = print $ head $ drop (10^8) $ cycle $ \ () -> xs
xs = [1..10^7]
```
This of course reintroduces the sharing I wanted to get rid of and restores the space leak.  Thus, the trick can only be used with `-fno-full-laziness`.

It might be worth mentioning that full laziness is a [disputed feature](https://gitlab.haskell.org/ghc/ghc/-/issues/8457).

Commenting
----------

_Write a comment by opening an issue in the [issue tracker](https://github.com/abel-blog/when-call-by-name-outperforms-call-by-need/issues)!_
