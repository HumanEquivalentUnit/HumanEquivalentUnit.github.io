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
APL uses the pipe for remainder,
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
is quite important and quite useful.

We'll use a boolean OR to combine the arrays of zero and one,
any position with a one in either array stays as a one kept.

Then we use the `/` function which is "replicate". 
It takes a control array of numbers on the left,
data on the right, and repeats each data item as many times
as the control number in the same position, says to.
`0 0 1 0 2/1 2 3 4 5` gives `3 5 5`

It's often used with zeros and ones to filter and drop,
zero meaning "replicate it no times" and one indicating "keep it".

```APL
      ⍝ Indicates multiples of three and multiples of five.
      (0=5|⍳10) ∨ (0=3|⍳10)
0 0 1 0 1 1 0 0 1 1 
      ⍝ pick out multiples of three or five from the integers to ten.
      ((0=5|⍳10) ∨ (0=3|⍳10)) / ⍳10
3 5 6 9 10 
      ⍝ Use the / operator which looks the same but is different, to sum them up
      ⍝ plus over the multiples of three and five in the integers to ten:
      +/ ((0=5|⍳10) ∨ (0=3|⍳10)) / ⍳10
33
```

and we have our test answer 23.

Oops, we have ten more than the desired answer, 
because the question says "numbers below ten", not "numbers to ten";
make the "integers to ten" into "integers to nine":

```APL
      +/ ((0=5|⍳9) ∨ (0=3|⍳9)) / ⍳9
23
```

[Try it on TryAPL.org](https://tryapl.org/?a=+/%20%28%280%3D5%7C%u23739%29%20%u2228%20%280%3D3%7C%u23739%29%29%20/%20%u23739&run)

Let's just split out the integers to nine into N,
to get rid of that repetition:

```APL
      N ← ⍳9
      +/ ((0=5|N) ∨ (0=3|N)) / N
23
```

Great, now I've introduced the basic patterns, I can start the blog post ;)

Can we get rid of the repeated code,
get the remainder of three and five in one operation?
Yes, with "all combinations of things on the left and things on the right"
called an outer product:

```APL
      ⍝ Get all combinations of three and five on the left,
      ⍝ integers to nine on the right, and combining with
      ⍝ remainder, to get two rows of output:
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

