---
layout: post
title: Arrays don't start at 0 /or/ 1
date: 2018-11-29
tags: [Programming-General,]
---

Programmers argue about whether arrays start at 1 or 0. (Read more about which languages use which [here on WikiPedia](https://en.wikipedia.org/wiki/Comparison_of_programming_languages_(array)))

I had the idea that this is not a fact about the array,
but a matter of choice about how we discuss it and use it.
Spiritual people say the world is not separated into many parts,
but putting names on a bit of the world makes us see is as separate.

We see `A, B, C, D` and it "is" the same thing for all of us,
there are four things and we can count them either `0, 1, 2, 3`
or `1, 2, 3, 4`.

When we talk about the array "starting at 0" that's what we see,
when we talk about `A` being "the 1st item" we see that instead.
See it as a *choice*, and then ask "why not make two ways to access it?",
a machine way and a human way:

    $array = @('A', 'B', 'C', 'D')
    $array.Item(0)            # -> 'A'
    $array.HumanItem(1)       # -> 'A'

The language need not choose, programmers can choose.
Then you can walk it either `0 to N-1` or `1 to N`.

PowerShell arrays start from index 0, 
but PowerShell array notation `0..4` makes five items `0, 1, 2, 3, 4`,
which is one too many. The array offset choice and the sequence notation are not in harmony,
so we often need to write the adjustment `0..(Length - 1)`.

We can't change `..` very easily, but we can add a HumanItem() method to an array,
and pretend it starts at 1:

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

You might guess what's coming - 
[Prof. Dijkstra's essay](https://www.cs.utexas.edu/users/EWD/transcriptions/EWD08xx/EWD831.html) about sequences,
where they start and end, and why it follows that starting at 0 makes sense.

And, he remarks:

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

Seems like `.Item()` vs `.HumanItem()` is a bad idea. 
Not a choice I want to make over and over, and prone to introducing mistakes.
(Thank goodness for iterators and `foreach ($x in $array) {}`
which takes care of the most common "go through all the items" case).

[Richard Feynman](https://www.youtube.com/watch?v=ga_7j72CVlc) was taught to separate
things from how people talk about them, "names are not knowledge",
but too much of that and you won't be able to communicate with people.

Which might be why "there are only two kinds of languages: the ones people complain about and the ones nobody uses" - Bjarne Stroustrup.

And why there are many LISP dialects and few Java dialects.

Put up with someone else's way of seeing the world (along with everyone else),
or make up your own (which nobody else uses).

The common suggestion "if you don't like it, change it" rings false.
If excluding someone from a group is rude, telling someone to exclude themselves also is.
Being unable to change something hurts, but being able to change anything
risks people isolating from each other, fragmentation, and loss of coherent direction.

