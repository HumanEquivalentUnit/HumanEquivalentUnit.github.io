---
layout: post
title: Arrays don't start at 0 /or/ 1
date: 2018-11-29
tags: [Programming-General,]
---

Programmers argue about whether arrays start at 1 or 0. (Read more about which languages use which [here on WikiPedia](https://en.wikipedia.org/wiki/Comparison_of_programming_languages_(array)))

I started this post thinking that it's not a fact about the array,
but a matter of choice about how we discuss it and use it.
Spiritual people say the world is not separated into many parts,
but putting names on a bit of the world makes us see is as separate,
and that is related.

We see the text `(A, B, C, D)` the same for all of us,
there are four things and we can count them. 

Both countings `(0, 1, 2, 3)` or `(1, 2, 3, 4)` can make sense,
when we talk about the array "starting at 0" we see the first counting style,
when we talk about `A` being "the 1st item" we see the second.

See it as a *choice*, and then ask if we talk about two valid ways to access it,
why not *make* two ways to access it?
The language need not choose, programmers can choose:

    $array = @('A', 'B', 'C', 'D')
    $array.Item(0)            # -> 'A'
    $array.HumanItem(1)       # -> 'A'

Then you can walk it with either sequence over the items: `0 to N-1` or `1 to N`.

PowerShell arrays start at 0, but we can fix it, 
add a HumanItem() method to an array, and pretend it starts at 1:

    PS C:\> $array = @('A', 'B', 'C', 'D')
    PS C:\> $array.Item(0)
    A
    PS C:\> ,$array | Add-Member -Name HumanItem -MemberType ScriptMethod -Value {
    >> param([ValidateScript({$_ -gt 0})][int[]]$Index)
    >> foreach ($i in $index)
    >>     {
    >>         $this[($i - 1)]
    >>     }
    >> }
    PS C:\> $array.HumanItem(1)
    A

Then write:

    PS C:\> $array[0..($array.Length - 1)]
    A
    B
    C
    D
    PS C:\> $array.HumanItem(1..$array.Length)
    A
    B
    C
    D

PowerShell arrays start from index 0,
and it has an array notation `0..4` to make a sequence of numbers,
but `0..N` to get all the items in an array actually makes five numbers: `0, 1, 2, 3, 4`,
if you want to get everything in the array you need to write the adjustment `0..(Length - 1)`.

You might guess what's coming - 
[Prof. Dijkstra's famous essay](https://www.cs.utexas.edu/users/EWD/transcriptions/EWD08xx/EWD831.html) about sequences,
where they start and end.

From that, Powershell is "doing it wrong".
Arrays should start at 0 and sequences like `0..N` should generate `0, 1, 2 .. (N-1)`.

But, I had forgotten this bit, he remarks:

> _Remark_ The programming language Mesa, developed at Xerox PARC,
> has special notations for intervals of integers in all four conventions.
> Extensive experience with Mesa has shown that the use of the other three 
> conventions has been a constant source of clumsiness and mistakes,
> and on account of that experience Mesa programmers are now strongly 
> advised not to use the latter three available features.
> I mention this experimental evidence —for what it is worth— because some
> people feel uncomfortable with conclusions that have not been confirmed 
> in practice. (End of Remark.)

And that's from 1982.

And instead of saying "it might be interesting to add two ways 
to reference items in an array", I'm no longer going with that.
Seems like `.Item()` vs `.HumanItem()` is a bad idea.
Not a choice I want to make over and over,
but worse - prone to introducing mistakes.

(Thank goodness for iterators and `foreach ($x in $array) {}`
which takes care of the most common "go through all the items" case).

---

On a related note, [Prof. Richard Feynman](https://www.youtube.com/watch?v=ga_7j72CVlc)
was taught to separate things from how people talk about them,
and in this video he tells a story about how "names are not knowledge",
but he also warns that going to far down that line of thinking leaves
you unable to communicate with people, because you don't know what anything is called.

Having two ways to index into an array would only make it harder to say which item was "number 1".

Put up with someone else's way of seeing the world (along with everyone else),
or make up your own (which nobody else uses).

