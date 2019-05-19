---
layout: post
title: APL - Project Euler 1
date: 2019-05-09
tags: [APL]
---

Many APL/J/K comments on the internet are "*here's my unreadable one-liner /splat/*", 
and it might be nice if people shared more of their thought processes.
Ideally experts would do it, but I'm a novice and I can only write my point of view.
I'm playing with the free NARS2000 version
([download here](http://www.nars2000.org/download/Download.html)).

The [first Project Euler problem](https://projecteuler.net/problem=1) is similar to FizzBuzz, 
it asks:

----

> If we list all the natural numbers below 10 that are multiples of 3 or 5, 
we get 3, 5, 6 and 9. The sum of these multiples is 23.
>
> Find the sum of all the multiples of 3 or 5 below 1000.

We need ten numbers, "integers to ten" is *iota ten*

```APL
      ⍳10
1 2 3 4 5 6 7 8 9 10 
```

and we need "divide by three, check if the remainder is zero".
APL uses the pipe for remainder, it looks like this:

```APL

      ⍝ 3|8 says "remainder after three goes into eight", answer is two.
      3|8
2
      ⍝ 3|⍳10 says the same, but remainders for three into all the integers to ten.
      3|⍳10
1 2 0 1 2 0 1 2 0 1 
      ⍝ 0=3|⍳10 asks where the remainder is zero, APL executes right to left;
      ⍝ see that the ones line up with three, six, nine, or the multiples of three.
      ⍝ and see the same pattern lines up multiples of five afterwards.
      0=3|⍳10
0 0 1 0 0 1 0 0 1 0 
      ⍳10
1 2 3 4 5 6 7 8 9 10
      0=5|⍳10
0 0 0 0 1 0 0 0 0 1 
```

This idea of an array of zeros and ones which line up with
the elements of another array by their position,
is often used in APL.

We'll use a boolean OR `∨` to combine the arrays of zero and one,
any position with a one in either array stays as a one kept.

```APL
      ⍝ Indicates multiples of three and multiples of five.
      (0=5|⍳10) ∨ (0=3|⍳10)
0 0 1 0 1 1 0 0 1 1 
```

Then we use the `/` function which is "replicate". 
It takes a control array of numbers on the left,
a data array on the right, and repeats each data item as many times
as the control number says to. Matching them up by position, e.g.
`0 0 1 0 2/1 2 3 4 5` gives `3 5 5`.

It's often used with zeros and ones to act as a filter,
zero meaning "replicate it no times (drop it)" and one indicating "keep it".

```APL
      ⍝ pick out multiples of three or five from the integers to ten.
      ((0=5|⍳10) ∨ (0=3|⍳10)) / ⍳10
3 5 6 9 10 
      ⍝ Use the / operator (which looks the same as the / function) to sum them up.
      ⍝ plus over the multiples of three and five, in the integers to ten:
      +/ ((0=5|⍳10) ∨ (0=3|⍳10)) / ⍳10
33
```

and we have our test answer 23.

Oops, we have ten more than the desired answer, 
because the question says "numbers below ten", not "numbers to ten";
let's make the "integers to ten" into "integers to nine":

```APL
      +/ ((0=5|⍳9) ∨ (0=3|⍳9)) / ⍳9
23
```

Now we have the test answer. [Try it on TryAPL.org](https://tryapl.org/?a=+/%20%28%280%3D5%7C%u23739%29%20%u2228%20%280%3D3%7C%u23739%29%29%20/%20%u23739&run)

Let's split out the integers to nine into a variable named N,
to get rid of that repetition:

```APL
      N ← ⍳9
      +/ ((0=5|N) ∨ (0=3|N)) / N
23
```

Great, now I've introduced the basic patterns, 
we can answer the prompt by changing N to the first 999 numbers,
and now I can start the blog post :)

----

Can we get rid of the repeated code,
get the remainder of three and five in one operation?
Yes, using "all possible combinations of things on the left, and things on the right",
called an outer product, to give one array with the multiples of three and five in it,
on separate rows:

```APL
      ⍝ Get all combinations of three and five on the left,
      ⍝ integers to nine on the right, and combining with
      ⍝ the remainder operation, to get two rows of output:
      N ← ⍳9
      0=3 5 ∘.| N
0 0 1 0 0 1 0 0 1
0 0 0 0 1 0 0 0 0
      ⍝ combine with the OR operation down the columns with ⌿
      ⍝ to get the same combinations we had earlier
      ∨ ⌿ (0=3 5 ∘.| N)
0 0 1 0 1 1 0 0 1 
      ⍝ replicate over N to pick out the multiples
      (∨ ⌿ 0=3 5 ∘.| N) / N
3 5 6 9      
      ⍝ plus over those, to sum them up
      + / (∨ ⌿ 0=3 5 ∘.| N) / N
23
```

Next, I upped N until it started to take a decently measurable time, 
in this case six nines, integers to 999999. 
The first one runs in 275-300ms.

The second one, with less duplication, runs in 390ms.
I'm surprised, I thought keeping things to one array would be faster.
My novice guess is that it's because APL keeps data in row-major order, 
meaning that in memory rows go 1111122222 so running along a row is fast,
the second approach to merge down "columns" is a lot more jumping around.

What if we go the other way about this,
instead of using division and remainder we could multiply by 3 and 5,
then sum up the results.

```APL
      ⍝ three times integers to nine, and five times them
      3×⍳9
3 6 9 12 15 18 21 24 27 
      5×⍳9
5 10 15 20 25 30 35 40 45
      ⍝ multiples of three and five, catenated into one array
      (3×⍳9), 5×⍳9
3 6 9 12 15 18 21 24 27 5 10 15 20 25 30 35 40 45 
      
      ⍝ But they go way above 9, so we need to filter them
      N←⍳9
      nums←(3×N), (5×N)
      (9≥nums)/nums
3 6 9 5 
```

Same result, which is good. Now we can increase N, 
but if we do, we get the wrong answer. 
Because multiples of three gets fifteen and multiples of five does too.
Let's use `∪` for unique, to remove duplicates:

```
      ⍝ Integers to 999, multiplied by three and five,
      ⍝ filtered to remove any above 999, and summed up.
      N←⍳999
      nums← ∪ (3×N), (5×N)
      +/ (999≥nums)/nums
233168
```

This works, but increasing it to 999999 for testing,
and it runs in 1.3 seconds. Not better.
Let's see if we can avoid generating the high numbers and throwing them away.

Instead of making all the multiples of three and five,
we can divide the cutoff by three,
and know that multiplying by three won't go beyond the cutoff.

```APL
      ⍝ How many multiples of three, won't take us past 19?
      19 ÷ 3
6.333333333
      ⍝ Integers to "one third of nineteen" is..
      ⍳ 19 ÷ 3
DOMAIN ERROR
      ⍝ oops, gotta round down here to use a whole number
      ⍳ ⌊ 19 ÷ 3
1 2 3 4 5 6
      ⍝ now multiply by three, show they don't go over nineteen
      ⍝ multiples of three, to nineteen
      3 × ⍳ ⌊ 19 ÷ 3
3 6 9 12 15 18

      ⍝ same with five, join the two together
      (3 × ⍳ ⌊ 19 ÷ 3) , (5 × ⍳ ⌊ 19 ÷ 5)
3 6 9 12 15 18 5 10 15

      ⍝ move N out into a variable of the cutoff integer,
      ⍝ not numbers up to a cutoff
      ⍝ and remove duplicates, and sum them
      N ← 999999
      +/ ∪ (3 × ⍳ ⌊ N ÷ 3) , (5 × ⍳ ⌊ N ÷ 5)
233333166668      
```

This runs in 300ms, which is nice. Best yet.

Instead of making duplicates and removing them,
I trial-and-error tested moving `∪` into the middle,
turns out this makes "unique" do set union instead,
which joins the two without duplicates in the first place,
instead of joining them with duplicates and later removing them.

```APL
      N←999999
      +/ (3 × ⍳ ⌊ N ÷ 3) ∪ (5 × ⍳ ⌊ N ÷ 5)
233333166668
```

This runs in 50ms. Way better!
I was pretty happy here,
and browsing the Project Euler forums where people share their solutions,
there were several in languages I couldn't read, 
and I wished people had added some explanation. 
That's where this blog post comes from. 
There were a few which added up the multiples of three and five,
then subtracted the multiples of fifteen - a good idea.

That looks like this

```APL
      N←999999
      (+/ 3 × ⍳ ⌊ N ÷ 3) + (+/ 5 × ⍳ ⌊ N ÷ 5) - (+/ 15 × ⍳ ⌊ N ÷ 15)
233333166668
```

and THAT runs in 2ms.
Less, I think; NARS2000 doesn't seem to make much distinction below 2ms.
I can keep increasing the value of N here and it keeps saying 2ms,
until ninety nine thousand trillion where it can't run.

Hopefully, that's a few approaches to a simple question explained,
and more evidence that "doing less work" is faster - 
but it's not obvious to me from a traditional programming language background,
what code results in "less work" in APL.