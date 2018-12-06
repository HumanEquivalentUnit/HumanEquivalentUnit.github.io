---
layout: post
title: PowerShell pipeline vs script vs wildcards
date: 2018-12-06
tags: [PowerShell,pipeline]
---

PowerShell lives in the middle of two worlds: 
writing code into scripts and running it,
and being an interactive shell to run programs and pipe output from one to another. 
It was designed to scale, one language which can be written in long-form to be clear and maintainable and re-usable,
or in short-form to be quick to type (aka "elastic syntax").

e.g. assigning a variable can be done by any of the following:

```
PS C:\> $lines = get-content -Path 'words.txt'
PS C:\> $lines = gc words.txt
or
PS C:\> get-content -path 'words.txt' | set-variable -name 'lines'
PS C:\> gc words.txt | sv lines

```

The usual discussion is about which is clearer and more readable (named parameters make intent clear to other people),
more stable (aliases like `sv` could change on different systems), or more performant (starting a pipeline takes time).
But, working in the shell, I often start typing before deciding what to do.
Typing some code and thinking about what will happen helps me decide,
then mid way through a line of code, I realise it needs to be different. 
From that position, all the debates about clarity and maintainability don't matter very much for one-line of code.
How easy is it to change the code I have, into the code I want?
Being able to chain variable assignment on the end is more fluid and less interrupting
than having to stop and edit the front of the line to add `$var =`.

Another common thing to decide mid-way through typing is that I need to extract some value,
e.g. get the length of all files in a folder, and only the length, e.g.:

```
PS C:\> Get-ChildItem | where-object { $_.Length -gt 0 } | Select-Object -ExpandProperty Length
PS C:\> gci -file | select -exp length
```

This is almost as short as it can go and switching styles to try `(gci).length` is usually better,
but in this case it returns the length of the array of files, instead of the length property of the members.

But then I learned about a way to use `ForEach-Object -MemberName` to lookup a property or call a method.
And it quietly suppresses errors so referencing the length of a directory won't fill the screen with errors.
And `foreach-object` has the default alias `%`

```
PS C:\> gci -file | foreach-object -membername Length`
PS C:\> gci |% length
```

But now we're in the syntax where `length` is an argument to foreach-object,
and it accepts wildcards to find the property:

```
PS C:\> gci | foreach-object -membername len*
```

The only catch is that the wildcard needs to resolve to one single property or method name,
otherwise you get a screenful of this:

```
PS C:\> gci | foreach -member l*
% : Input name "l*" is ambiguous. It can be resolved to multiple matched members. Possible matches include: LinkType
Length LastAccessTime LastAccessTimeUtc LastWriteTime LastWriteTimeUtc.
```

Which is intimidating, but helpful - it tells exactly how the match is going wrong.
Part of the fun of using `|foreach -member ___` is playing around to find a wildcard 
pattern which is short enough. You can even play for fun wildcards instead, e.g.
`gci | % LastW*Time` is clearly going to end up as `LastWriteTime`, but what is
`gci | % F*L*A*M*E` going to be? Or `$hash = @{'a'=1;'b'=2}; $hash |% g*u*m*r*a*t*` ?

I play some CodeGolf, the game of writing code as short as possible, 
which also makes the code unreadable. It's no use in the real world, 
as per the famous quote "*I have made this longer than usual because I have not had time to make it shorter.*", 
writing really really short code takes forever.
But the techniques for short code can help, it's much nicer if you can proof a concept or test an idea
in a few short commands, rather than half a page of code.
It makes it more likely that you *will* explore an idea or test something,
because you can progress quickly,
try five approaches before someone else has finished trying one.

So the exploration of short commands and wildcards for parameters, 
trying them and seeing where they are allowed, led me to `get-variable`,
this is the code/cmdlet way of doing:

```
PS C:\> $x
PS C:\> get-variable -name 'x' -valueonly
```

I have very little idea who uses this as the main approach to variables,
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
in [issue 8407](https://github.com/PowerShell/PowerShell/pull/8407).

Whether it will get "fixed" is now a matter of backwards compatibility.
Anyone using it the way it works now, risks having their scripts break.
Since there is already a split between PowerShell 5 and 6,
the team is willing to have some code change, but not everything.
They try to judge if the break is minor, or if the long term benefit is worth it.

We'll see.

Still. Try to use both styles of "classic code" and "cmdlets and pipelines" because
more tools means you can do more things and solve more problems. 
And write short code in the shell, then each step in a problem takes less effort,
so the whole problem does, so it's solved sooner.
It doesn't matter too much if it's unreadable - most of your command history won't be read.
