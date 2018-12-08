---
layout: post
title: Speed tweaking Advent of Code Day 8
date: 2018-12-08
tags: [PowerShell,performance]
---

The [Advent of Code Day 8](https://adventofcode.com/2018/day/8) puzzle starts with a single line of integers,
and asks you to parse a tree out of it. 
[My code is here](https://github.com/HumanEquivalentUnit/AdventOfCode2018/blob/master/2018-12-08-PowerShell-p1-and-p2.ps1),
and [my input data is here](https://github.com/HumanEquivalentUnit/AdventOfCode2018/blob/master/2018-12-08-data.txt);
this blog post is about how I tweaked it from 1.1 seconds runtime to 0.205 seconds runtime.

Click for a [1 min 40 second video of my undo/redo buffer from the start to the end of all of this](https://github.com/HumanEquivalentUnit/AdventOfCode2018/blob/master/2018-12-08-PowerShell-coding.mp4).

<!--more-->

The testing code is `measure-command { .\code.ps1 | out-default }` and that shows both the code result, 
and the timing data. It's not very accurate - use System.Diagnostics.StopWatch for that - 
but run it two or three times after a change, and it's good enough to show gain or loss.

At this point, I had written it with a stack, a `switch` and some lists and it was working,
running over an input of ~17,000 numbers, getting results for both puzzle parts in ~1.1 seconds.

After spending a while tweaking, I got it down to ~205ms. 

Then I hit `Ctrl-Z` a lot in PowerShell ISE to undo the changes, 
and documented them below. This is a reverse log of trial-and-error 80% speedup.

----
### Next
The core of my code was a while loop until the string ended, 
and a `switch()` based state machine which was going to have states like "starting a new node",
"starting a child node", "getting metadata", "updating the tree", "returning up a level".
It ended up only needing "start of a node" and "end of a node".

So what handles the states in the state machine? I had `strings` for states, like so:

```powershell
$parserState = 'NewNode'

switch($parserState) {
    'NewNode' {
    }
    'MetaDataCleanup' {
    }
}
```

and I thought that string comparisons might have to be done from start to end,
and PS does them case insensitively so there might be a lot of work happening behind the scenes.
`enums` have integer values, and a number comparison might be much faster than a string comparison:

```powershell
enum state {
    NewNode
    MetaDataCleanup
}

$parserState = [state]::NewNode

switch($parserState) {
    ([state]::NewNode) {
    }
    ([state]::MetaDataCleanup) {
    }
}
```

Result: worse! much worse.
Removing the strings and enums, and going to integers with my own code,
using a simple `$stateNEWNODE = 0; $stateMETADATACLEANUP = 1` 
for states was significantly faster than strings or enums, ~100-200ms, I think.

----
### Next
When parsing a new node I make a hashtable for it, 
with lists to hold the childNodes and MetaData values:

```powershell
               @{ 
                ...
                childNodes         = [List[psobject]]::new()
                metaData           = [List[psobject]]::new()
                ...
               }
```

I wanted to make that conditional, to save the cost of initialization, so I tried something like this:

```powershell
            $childNodeList      = if ($numChildNodes      -gt 0) { [List[psobject]]::new() } else { $null }
            $metaDataList       = if ($numMetaDataEntries -gt 0) { [List[psobject]]::new() } else { $null }
```

Result: SCREEN FULL OF EXCEPTIONS

Scratching my head long time, tried several variations, 
until I realised the `if` was enumerating the values from the empty list,
and assigning `$null`. I would have to use the trick of putting `,` 
in front of it to make a 1-item-array wrapper so the unrolling works on that and leaves the list alone:

```powershell
            $childNodeList      = if ($numChildNodes      -gt 0) { ,[List[psobject]]::new() } else { $null }
            $metaDataList       = if ($numMetaDataEntries -gt 0) { ,[List[psobject]]::new() } else { $null }
```

Result: no benefit, maybe slower, and ugly code. Not sure if memory use improved, but code change reverted.

----
### Next
In part 2 adding up the metadata values (sum a collection of ints):

```powershell
                $currentNode.value = ($currentNode.metaData | measure-object -Sum).Sum
```

I wanted to avoid the pipeline setup cost, 
and cmdlet name resolution and initialization costs,
with this fast loop:

```powershell
                $localSum = 0; foreach ($m in $currentNode.metaData) { $localSum += $m }
                $currentNode.value = $local
```

Result: bigger speedup than I expected, worth doing even though the code looks uglier.
Could certainly be spread onto more lines, 
but makes me wish for Python's `sum(Iterable)` construct.

----
### Next
The state machine was down to two states: "state 0, new node is starting"
and "state 1, node ending". Code like this:

```powershell
            $parserState = $stateMETADATACLEANUP
            if ($nodeSkeleton.numChildNodes -gt 0)
            {
                $parserstate = $stateNEWNODE
            } else
            {
                $parserState = $stateMETADATACLEANUP
            }
     
     
           # [..] in the cleanup state
     
     
            if ($parent.childNodes.Count -lt $parent.numChildNodes)
            {
                $parserState = $stateNEWNODE
            }
            else
            {
                $parserState = $stateMETADATACLEANUP
            }
```

This use of if/else tests every time through the loop was bothering me. 
Eventually I realised that they're not really states, 
they're counters for how many child-nodes still need parsing below,
before we can get to this level's metadata. All that became this:

```powershell
            $parserState = $nodeSkeleton.numChildNodes
 
            # [..] and this
            
            $parserState = $parent.childNodes.Count - $parent.numChildNodes
 ```

Now it falls to "state 0" when there are no childnodes at all, or no more remaining, 
and "state N" (switch 'default') when there are some new nodes to start.
(I think I swapped the switch blocks around as well at this point).

Result: pleasingly less code, some speed improvement, quite small, not really worth doing.

----
### Next
Reading the metadata after all the childNodes are done, 
I had a fast loop adding to a Generic List:

```powershell
            #
            # previously  # 1..$currentNode.numMetadataEntries | ForEach-Object {
            #
            foreach ($counter in 1..$currentNode.numMetadataEntries)
            {
                $currentNode.metaData.Add($inputNums[$i++])
            }
```

but with a bit of thinking about how `$i` would move during this,
the loop can go, and be replaced with a multi-index lookup and direct array assignment:

```powershell
            $currentNode.metaData = $inputNums[$i..($i+$currentNode.numMetadataEntries-1)]
            $i += $currentNode.numMetadataEntries
```

and lower down in the code / earlier on in the exeuction when setting up a new node, 
`metaData = [List[psobject]]::new()` can become `metaData = $null` saving setup time and memory.
Result: pleasingly large gain, can't remember how much, but ~30ms region ish.

----
### Next
Realised I was calculating the `$localSum` sum-of-metadata-for-a-node twice. Once from Part 1 of the puzzle, again for part 2.
Removed that duplicate work. It helped a little.

----
### Next
In part 2 of the puzzle, there is a lookup into the childNodes 1-indexed,
and a sum of those values.

Earlier it had been a loop with a check:

```powershell
                    $m--    # 1-indexed here
                    $node = $currentNode.childNodes[$m]
                    if ($node)
                    {
                        $currentNode.value += $node.value
                    }
```

and I had removed the `if` because PS will handle reading outside the bounds of an array,
and casts $nulls to int 0. At this point it was already a fast `foreach(){}` loop like so:

```
                foreach ($m in $currentNode.metaData)
                {
                    $node = $currentNode.childNodes[$m-1]
                    $currentNode.value += $node.value     # might be $null, but PS is fine with that
                }
```

(It occurs to me now, that implicit casting is a bit slow, 
it might be faster if I hadn't done this, and used if/else).

But I hoped to use a multi-lookup with `childNodes[$currentNode.metaData]` instead of a loop in my code,
make the PS Engine loop in faster C# code behind the scenes. To get around the need to `$m-1` every time,
I shunted the nodes to the side by 1 and used this:

```powershell
                $currentnode.childNodes.Insert(0, 0)
                foreach ($v in $currentNode.childNodes[$currentNode.metaData].value) { $currentNode.value += $v }
                $currentNode.childNodes.RemoveAt(0)
```

Result: measurable, pleasing improvement; from memory in the ~25ms region.

----
### Next
The code to start parsing a new childNode makes a hashtable and pushes it on the stack.
Then does a property lookup to reference the number of childnodes to find which state to go to next.
I wanted to avoid that property lookup, so I inlined the state assignment into the hashtable literal.

```powershell
            $nodeSkeleton = @{
                numChildNodes      = $inputNums[$i++]
                numMetadataEntries = $inputNums[$i++]
                childNodes         = [List[psobject]]::new()
                metaData           = $null
                value              = 0
            }
            $stack.Push($nodeSkeleton)
            
            $numNodesAtThisLevelToParse = $nodeSkeleton.numChildNodes
```

turned into:

```powershell
            $stack.Push(@{
                numChildNodes      = ($numNodesAtThisLevelToParse = $inputNums[$i++])
                numMetadataEntries = $inputNums[$i++]
                childNodes         = [List[psobject]]::new()
                metaData           = $null
                value              = 0
            })
```

Result: from memory, mild improvement. 

----
### Next

When I started I had `[system.collections.generic.List[psobject]]` all over the place, 
and was annoyed with typing it. [Joel Bennet / Jaykul](https://twitter.com/Jaykul) in the PowerShell Slack chatroom
reminded me that `using system.collections.generic` can go at the top,
and then each one can be shortened to `[list[psobject]]`.
At this point in my code I had cut down the use of it quite a bit, 
and wondered if this namespace lookup shortname-to-fullname might cause a measurable delay.
It does in a lot of PowerShell, resolving `where` to `where-object` takes time, for example.

I tried changing them to `[system.collections.generic.list[psobject]]`.

No performance change that I could tell, but I left it in.

----
### Next

The original data reading was `Get-Content .\data.txt -raw` and then a line split and cast to int.
I thought it would definitely be faster avoiding the cmdlet and using .Net file reading.
But .Net has a different idea of "the current working directory", and I wanted to handle that.

```powershell
$lines     = Get-Content .\data.txt -raw
$inputNums = [int[]]($lines.Split(" `r`n"))
```

Changed to a long and ugly line such as

```powershell
$inputNums = [int[]][System.IO.File]::ReadAllText((Get-Location).Path + '\data.txt')).Split(" `r`n")
```
But it was no faster. Not much surprise as I left another cmdlet call in there.
While writing this, I've found `$pwd.Path` to get the PS path and avoid that cmdlet,
but I can't tell if it makes any change one way or the other.
Change reverted because it's too ugly this way.

----
### Next

Here's when I realised the "state machine" had boiled down to two states, 
which could be simplified to an `if/else`.
I swapped this `switch` pattern:

```powershell
switch ($numRemainingChildNodes)
{
    0 {
    }

    default {
    }
}
```

for this:

```powershell
if ($numRemainingChildNodes -gt 0)
{}
else
{}
```

I wasn't really expecting a performance change because switch is a builtin keyword, 
just wanted it shorter and cleaner. Surprisingly, it did help ~10-15ms.

----
### Next
Once the stack is empty and we're nearing the end,
I need to avoid calling `.Peek()` on it, because that throws an exception.
Code started as:

```powershell
        # empty stack means $current is the finished root node and we're done.
        if ($stack.count -gt 0)
        {
            $parent = $stack.Peek()
            $parent.childNodes.Add($currentNode)
                
            $numRemainingChildNodes = $parent.childNodes.Count - $parent.numChildNodes
        }
```

But this means doing the `if` test for every node end, 
when there's only the root node which will empty the stack.
How wasteful.
First try was just to comment out the `if () {}` block and let it throw an exception,
and deal with seeing the error message.
Would saving the test cost many times win over the large cost of raising and propagating an exception?
No it wouldn't, letting it throw an exception makes it ~20-35ms slower.

I looked around for another way to do the test than reading and comparing the count, 
hoping to find `$stack.IsEmpty` but nothing looked good.

Second try was to add a fake-node to the stack when initializing it,
now I can .peek() the stack and it will never be empty.
Avoids the test on every node and the exception.

```powershell
# at the very top, push a buffer node onto the stack.
$stack.Push(@{childNodes=[System.Collections.Generic.List[psobject]]::new()})

# now at the end, this will always succeed, no need for a special case
        $parent = $stack.Peek()
        $parent.childNodes.Add($currentNode)
```
This helped by ~5ms.

----
### Next

Somewhere in here I learned that implicit casting is slower.
For a long time I have advocated `if ($x.Thing -eq $true)` -> `if ($x.Thing)`
and in a similar way `if ($x.Count -gt 0)` -> `if ($x.Count)`.
PS can and will do the implicit conversion to `[bool]` so let's use it.

But the operator `-gt 0` seems to be faster than `[bool]$x.Count`.
There are reasonably complex rules for what PS objects are truthy vs. falsey,
so perhaps there is overhead in going through those states, 
and "number -gt 0" can ultimately boil down to a single CPU instruction.
(Not sure if it actually does, but it could).

----
### End

Final runtime, in ISE, with several runs, varies around 205ms. Down to ~197ms and up to ~250ms.

I didn't keep a log of these at the time, so this is dependent on PowerShell ISE's undo buffer.
Stepping back through my changes, *most* of them were rewriting and shuffling comments around,
rewriting variable names, and laying out code.

(All of the numbers are imprecise. Numbers like ~5ms and ~25ms are only rough indicators
of effect size, as I percieved it, on my system. Trying to recreate a couple, ISE seems to 
run this script slightly faster than PowerShell.exe).
