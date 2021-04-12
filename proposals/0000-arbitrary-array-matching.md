# Arbitrary-length Array Matching with `Rest`

* Proposal: [HXP-NNNN](NNNN-rest-matching-arrays.md)
* Author: [Evan Montembeault](https://github.com/montibbalt)

## Introduction

Support variable capture when pattern matching arbitrary-length Arrays, via the
`...` and `...rest` syntax.

## Motivation

Curently, Haxe's pattern matching can only capture variables from Arrays with a
specific length, i.e. it can capture from Arrays with *exactly* N elements, but
not from Arrays with *at least* N elements.

One basic workaround is to match a wildcard, possibly add a guard on the Array
length, and manually assign vars. Drawbacks of this workaround include:

* The manually captured vars can't be used in the case expression (guards, etc)
* It causes the same boilerplate that Array matching already removes
* Using guards can cause `Unmatched patterns: _` even if all cases are covered

## Detailed design

Haxe 4.2 added "Rest" arguments for functions, with the following
[documentation](https://github.com/HaxeFoundation/haxe/blob/844cbe6e907418a950d4c685a06743546c01afb1/std/haxe/Rest.hx#L8):

```
Should be used as a type for the last argument of a method, indicating that
an arbitrary number of arguments of the given type can be passed to that method.
```

I propose that Haxe's `...` and `...rest` syntax could intuitively simplify
arbitrary-length Array matching.

* `...` could specify that the Array has 0 or more additional elements
* `...rest` could capture any additional elements in a new Array named `rest`

```haxe
function foo(ints:Array<Int>):String {
    return switch ints {
        // 1. Nothing would change or break with existing Array patterns
        case null | []:
            'empty';
        case [x]:
            'only $x';
        case [x, y]:
            'only $x and $y';

        // 2. It can get ugly if we don't know the exact Array length,
        // especially if we need guards on Array elements. For example:
        case _ if(ints.length >= 2 && ints[0] == ints[1]):
            var x = ints[0];
            var y = ints[1];
            '$x equals $y';

        // But `...` could match >= N elements more clearly:
        case [x, y, ...] if(x == y):
            '$x equals $y';

        // 3. Capturing the "tail" of an Array is just as awkward
        // Consider the following:
        case _ /*if(ints.length > 0)*/:
            var head = ints[0];
            var tail = ints.slice(1);
            '$head :: $tail';

        // Versus the `...rest` syntax already used for functions:
        case [head, ...tail]:
            '$head :: $tail';

        // Wildcard matches with guards like (2.) can require this, perhaps rest
        // patterns could eliminate that? It wouldn't make this worse either way
        case _: throw 'impossible but sometimes required pattern';
    }
}

Assert.equals('empty',        foo(null));
Assert.equals('empty',        foo([]));
Assert.equals('only 0' ,      foo([0]);
Assert.equals('only 0 and 1', foo([0, 1]));
Assert.equals('0 equals 0',   foo([0, 0, 2]));
Assert.equals('0 :: [1, 2]',  foo([0, 1, 2]));
```

Other languages support similar patterns (at least with Lists). It is so common
in Haskell for example, it's in the beginner `quicksort` example:

```haskell
-- https://wiki.haskell.org/Introduction#Quicksort_in_Haskell
quicksort :: Ord a => [a] -> [a]
quicksort []     = []
quicksort (p:xs) = (quicksort lesser) ++ [p] ++ (quicksort greater)
    where
        lesser  = filter (< p) xs
        greater = filter (>= p) xs
```

Versus the same implementation in Haxe if this is accepted and doesn't require
an "impossible" case:

```haxe
// A `...rest` pattern feels natural here
function quicksort(arr:Array<Int>):Array<Int> {
    return switch arr {
        case []: [];
        case [p, ...xs]:
            var lesser = xs.filter(y -> y <= p);
            var greater = xs.filter(y -> y > p);
            quicksort(lesser).concat([p]).concat(quicksort(greater));
    }
}
```

## Impact on existing code

This is a specific quality-of-life addition to pattern matching syntax and
should not impact existing code.

## Drawbacks

It could become confusing whether an exact Array length is required or not.

This could make the order of patterns even more significant.

There could also be performance implications that are hidden from the developer
if e.g. Array slicing is used to capture the "Rest" of the Array.

## Alternatives

Perhaps macros might also enable this, but I think that could quickly cause
macro-in-macro issues?

## Opening possibilities

More advanced patterns could be unlocked by allowing the syntax in other
positions.  
For example, `case [a, ..., c]:` could capture the first and last elements of an
Array with length >= 2.

With more consideration, this might enable a similar type of matching on
`Iterable`.

## Unresolved questions

