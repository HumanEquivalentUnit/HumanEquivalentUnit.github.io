---
layout: post
title: PowerShell pipeline vs script vs wildcards
date: 2018-12-06
tags: [PowerShell,pipeline]
---

Some background and context for how [Joel's PowerShell issue 8407](https://github.com/PowerShell/PowerShell/pull/8407) was found and fixed.

PowerShell lives in two worlds: scripts, and interactive shell. Scripts tend to look like other programming languages with loops and functions and error handling, they want to be clear and maintainable. 
Shell commands look like other shells, with commands and parameters and pipelines, they want to be something you could reasonably type and compose together without too much "programming" overhead.
PS has the idea of "elastic syntax", that one language can be written short-form or long-form, in a programming style or a shell style. (Downside: it has a *ton* of syntax).

e.g. assigning a variable can be done by these:

```
PS C:\> $lines = get-content -Path 'words.txt'
PS C:\> $lines = gc words.txt
or
PS C:\> get-content -path 'words.txt' | set-variable -name 'lines'
PS C:\> gc words.txt | sv lines

```

The usual discussion is clarity and maintenance (full names and parameters make intent clear),
or stability (aliases like `sv` could change on different systems), 
and in the shell people talk about how much they have to type.

But in the shell I start typing before fully deciding what to do,
writing code helps me decide what to do,
then mid-way through a line of code,
I realise it needs to be different.
At this point, typing is not my concern, but disruption is.
How easy is it to change the code I have, into the code I want?
Part of this is about tooling such as [PSReadLine](https://github.com/lzybkr/PSReadLine)
which handles keyboard shortcuts for jumping the cursor around by word, by line, etc.
But another part of it is being able to rework one style of code into another,
chaining variable assignment on the end with `|sv tmp` is more fluid and less interrupting
than having to stop and edit the front of the line to add `$tmp =`.

---

Another common thing to decide mid-way through typing is that I need to extract some value,
e.g. get the length of all files in a folder:

```
PS C:\> Get-ChildItem | where-object { $_.Length -gt 0 } | Select-Object -ExpandProperty Length
PS C:\> gci -file | select -exp length
PS C:\> (gci).length     # this one switches styles. But it doesn't work for .Length.
```

The second is almost as short as it can go; switching styles to try `(gci).length` is usually better,
but in this case it returns the length of the array, instead of the `.Length` property of the members.

But then I learned about a way to use `ForEach-Object -MemberName Length` to lookup a property or call a method.
And it quietly suppresses errors so referencing the length of a directory won't fill the screen with errors.
And `foreach-object` has the default alias `%`, so it became a staple shell pattern for me, overnight:

```
PS C:\> gci -file | foreach-object -membername Length`
PS C:\> gci |% length
```

Here, `length` is an argument to foreach-object,
and that cmdlet accepts wildcards to find the property:

```
PS C:\> gci | foreach-object -membername len*
```

Which is bizarre because the wildcard must to resolve to one single property or method name,
otherwise you get a screenful of this:

```
PS C:\> gci | foreach -member l*
% : Input name "l*" is ambiguous. It can be resolved to multiple matched members. Possible matches include: LinkType
Length LastAccessTime LastAccessTimeUtc LastWriteTime LastWriteTimeUtc.
% : Input name "l*" is ambiguous. It can be resolved to multiple matched members. Possible matches include: LinkType
Length LastAccessTime LastAccessTimeUtc LastWriteTime LastWriteTimeUtc.
[..]
```

Which is intimidating, but helpful - it tells you exactly how the match is going wrong.
(Why it supports a wildcard here, when it cannot return multiple matches, I don't know).
Part of the fun of using `|foreach -member ___` is because of the feedback loop of this error message,
playing around to find a wildcard pattern which is short but only matches one name.
You can play for fun wildcards instead of short ones, e.g.
`gci | % LastW*Time` is clearly going to end up as `LastWriteTime`, but what is
`gci | % F*L*A*M*E` going to be? Or `$hash = @{'a'=1;'b'=2}; $hash |% g*u*m*r*a*t*`
or `gci -file | group extension |% ????`?

(Another neat one is `gc file.txt | sort *` to sort strings by length).

I play some CodeGolf, the game of writing code as short as possible, 
which happens to make the code unreadable (that's not the point of it).
It's no use in the real world, like the famous quote
"*I have made this longer than usual because I have not had time to make it shorter.*", 
writing really really short code takes ages.
But the techniques used for short code are useful in the real world,
it's much nicer if you can make a proof of concept or test an idea
in a few short commands, rather than half a page of code.
If you can use programming syntax and command-syntax and mix them up,
you can save deleting and rewriting code differently.
It makes it more likely that you *will* explore an idea or test something,
because you can make progress quickly with low effort,
try three things in the time it would have taken you to try one.

----

So explorating short commands and wildcards, 
trying them and seeing where they are allowed, 
is how I came to be up looing at `get-variable`,
which is the code/cmdlet version of just `$x`:

```
PS C:\> $x = 1
PS C:\> $x
1
PS C:\> get-variable -name 'x' -valueonly
1
```

I have little idea who uses this as the main approach to variables,
but it also exists as the alias `gv` and it accepts wildcards for the name.
Now:

```
PS C:\> $someVeryLongNameHere = 1,2,3,4,5
PS C:\> $someVeryLongNameHere = 1,2,3,4,5
1
2
3
4
5
PS C:\> $someVeryLongNameHere | measure-object -sum | select-object -expandproperty sum
15
PS C:\> gv some* -valueonly
1
2
3
4
5
PS C:\> gv some* -valueonly | measure -sum
measure : Input object "System.Object[]" is not numeric.
At line:1 char:14
```

*recordscratch*

And this is the story all about how I tripped over the fact that two ways to get the value of a variable
don't behave the same. `$x` lists the values of a container (e.g. an array) into the pipeline, one at a time.
Get-Variable outputs the array as a single whole item down the pipeline.

I commented about this on the PowerShell Slack channel,
and Joel ([Vexx32](https://github.com/vexx32) guessed immediately what is happening,
jumped into the source code of the `Get-Variable` cmdlet and confirmed it.
There is a way for cmdlets to write their output to the pipeline,
and they have a choice of whether to trigger enumeration/unrolling of the content or not.

And because he's a good egg, he fixed it and submitted a 5-character fix to the PowerShell project, 
in [issue 8407](https://github.com/PowerShell/PowerShell/pull/8407). This change:

```
-     WriteObject(matchingVariable.Value);
+     WriteObject(matchingVariable.Value, enumerateCollection: true);
```

Whether it will be accepted is now a matter of backwards compatibility.
Anyone using it the way it works now risks having their scripts break.
But there is already a compatibility break between PowerShell 5 and 6,
the team is willing to have some behaviours change, but not everything.

We'll see.

Still. Use both styles of "classic code" and "cmdlets and pipelines" because
better grasp of PS means you can tackle more problems, with less work. 
And write short code in the shell, so each step in a problem takes less effort,
so the whole problem is easier, you can explore more approaches.
It doesn't matter too much if it's unreadable - most of your command history won't be read.
