---
layout: post
title: Advent Of Code 2018 - Optimizing Day 1 
date: 2018-12-18
tags: [PowerShell,Optimization]
---

Taking Day 1 code from 3.1 seconds to ~63ms.

As I left it, the code to Part 2 read an array of integers from a file,
repeatedly adding them up until it found one it had seen before. 
It already used a dictionary for a fast lookup, 
and `.foreach{}` for a fast loop, it took over 3 seconds and looked like this:

```powershell
$nums = [int[]](get-content .\data.txt)
$lookup=@{}
$current=0
while ($true) { 
    $nums.foreach{ $current += $_;   if ($lookup.ContainsKey($current)) {break}; $lookup[$current]++; } 
}
$current
```

(I ditched the commandlets and called `[System.Io.File]::ReadAllLines()` and `[Linq.Enumerable]::Sum()` for Part 1, ~30ms saving).

Changing `$nums.foreach{}` to `foreach ($n in $nums)` means the `break` needs adjusting with 
[a label on the outer while loop](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_break?view=powershell-6).
This loop change makes it twice as fast, down to around 1.5s.

Switching the hashtable for a `[Collections.Generic.Dictionary[int,int]]` helps a lot;
generic collections don't have overhead of boxing/unboxing integers.
That brought it down to ~67-70ms, 
but `[System.Collections.Generic.HashSet[int]]` looked interesting. 

Instead of doing a two-stage Dictionary Add/ContainsKey, 
HashSet.Add() returns True for a first time addition,
False if the data was already present, 
so it's now a single step store-and-test-if-previously-seen.
That saves 5-10ms.

Outputting two values to the pipeline, surprisingly costly. 
5-10ms saved by using `[System.Console]::WriteLine`,
presumably avoiding the output formatters.

This is how it looks now, both parts running in 2% of the time of the earlier code:

```powershell
$nums = [int[]][system.io.file]::ReadAllLines('d:\aoc\2018\1\data.txt')

# Part 1
[System.Console]::WriteLine([System.Linq.Enumerable]::Sum($nums))

# Part 2 - keep summing until a repeat is seen.
$lookup = [System.Collections.Generic.HashSet[int]]::new()
$runningTotal = 0
:outerLoop while ($true) { 
    foreach ($n in $nums) { $runningTotal += $n; if (-not $lookup.Add($runningTotal)) { break outerLoop } } 
}
[System.Console]::WriteLine($runningTotal)
```

(My measuring code is `measure-command{ .\Day1.ps1 | Out-Default}` which is not the most precise, 
but good enough to show cutting 90% of the runtime off).
