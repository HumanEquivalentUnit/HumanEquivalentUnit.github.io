---
layout: post
title: Reading N characters from the end of a text file
date: 2018-12-01
tags: [PowerShell,Programming, Unicode]
---

#### Someone

asked why PowerShell `Get-Content -Tail` only works with the last few lines of a file,
and why it cannot read the last N characters from the end of a text file with no line breaks.
e.g. if they have a 2Gb file which is "a single line", it's no help.
They wanted to get the last 100 characters quickly.

That's easy enough, I thought. (TL, DR. It wasn't).

<!-- more -->

---

#### You can use .Net to read the last 20 bytes from a file easy enough:

    $numBytes = 20
    $file = Get-Item -Path .\bigfile.txt

    # Open file as a stream, 
    # and seek back from the end.
    $stream = [System.IO.File]::OpenRead($file.FullName)
    [void]$stream.Seek(-$numBytes, [System.IO.SeekOrigin]::End)

    # Read some bytes from it into a buffer,
    # and decode them into text
    $buffer = [byte[]]::new($numBytes)
    [void]$stream.Read($buffer, 0, $numBytes)
    $stream.Close()
    
*And* you can turn them into a string, easy enough:

    [System.Text.Encoding]::ASCII.GetString($buffer)
    
Which is alllmost fine. Oh btw, what's that over there...

---

#### Hey everyone, it's UNICOOOOOODE!

If you learned to code, you probably did a lot of exercises like 
"reverse a string", "treat a string as a list of characters", 
"do ROT-13 encryption on a string", 
and these are fine for learning about loops and indexing and code,
but terrible for imprinting bad ideas about writing and text.

They will have introduced the idea that `"a"` is a character,
it somehow has the value `97` because computers can only work with numbers,
and that's related to ASCII.

ASCII is "the" standard which covers what letter becomes what number,
and everyone agrees so that people can convert chars to numbers,
other people can convert them back again and get the same chars.
Several chars in a row is a string.

If you are also monolingual English speaker this is fine.
My classic understanding looked like this:

|          | Classic |
|----------|---------|
|          | "a"     |
|          | -> ASCII|
| byte(s): | 97      |

There are other character sets used by French people and the like,
but they just choose numbers for `é` which is simple enough,
just agree which encoding you're using.

Then I read a bit about Unicode, and my understanding changed to this:

|          | Classic | Basic Unicode |
|----------|---------|---------------|
|          | "a"     | "é"           |
|          | ASCII   | Unicode?      |
| byte(s): | 97      | 233           |

It does the same thing in the same place in the same way but it covers more characters.
Simple, useful, but incorrect. `é` just happens to fit inside the value of a byte, 0-255. 
What about all the thousands of characters which don't? Like `ă` with value 259?

Turns out that Unicode is more than a simple mapping of one letter to one byte,
it has "multi-byte" characters! 
The original choices were 1, 2, 4 bytes per character, a tradeoff.
1 byte cannot represent most characters but it can represent English just fine.
2 or 4 bytes can represent most characters, but English speakers don't need that
so 50% or 75% of the storage and memory space and processing time is wasted for English text.

Then UTF-8 came along which is "1-4 bytes, varying, as few as necessary".
See this short [Tom Scott YouTube video](https://www.youtube.com/watch?v=MijmeoH9LT4) 
on the topic of UTF-8 and how great it is.

In the Powershell world you mostly see:

 - UTF-16 (aka UCS2-LE), which uses two bytes per character. Windows and .Net often use this.
   - `PS C:\> [System.Text.Encoding]::Unicode.GetBytes("ă")` -> `3, 1`
 - UTF-8, which Linux often uses, and PowerShell tries to.
   - `PS C:\> [System.Text.Encoding]::UTF8.GetBytes("ă")` -> `196, 131`
 
So my new understanding of Unicode became something like this:

|          | Classic | Basic Unicode | Next Unicode |
|----------|---------|---------------|--------------|
|          |         |               | "ă"          |
|          |         |               | Unicode?     |
|          | "a"     | "é"           | Codepoint U+0103|
|          | ASCII   | Unicode?      | -> UTF8  ->  |
| byte(s): | 97      | 233           | 196 131      |

and that was fine for a long time. 
This understanding is just about enough to fight Unicode problems, 
which often happen when text was encoded to bytes one way,
then decoded back to text another way. Let's see a couple:

Smartquotes like `“` run through UTF8 then back through UTF16-LE:

    PS C:\> $bytes = [system.text.encoding]::UTF8.GetBytes('“')
    
    PS C:\> [system.text.encoding]::Unicode.GetString($bytes)
    胢�

Text through Unicode then back through UTF8:

    PS C:\> $bytes = [system.text.encoding]::Unicode.GetBytes('test')
    
    PS C:\> [system.text.encoding]::UTF8.GetString($bytes)
    t e s t 

There's a few common versions of these, 
and a few more when considering HTML like `£` showing up wrong, 
and this level of understanding has helped me imagine what's going wrong and fix it on many occasions.

---

#### Back to the end of the file

So we don't know what the text file contains, 
but the maximum encoding is 4 bytes per character,
then reading N characters from the end of the file simply means reading 4*N bytes,
converting to string, and if that was too many, trimming down a bit.

Easy. And it works.

It just means a bit of error checking, because for a file of 5 bytes, 
asking for 3 chars and trying to read 12 drops back before the file begins.

And it means a bit of guessing what the encoding of the file is,
but you could always just make the user tell you, 
or guess, or default to UTF-8 which is what a lot of tools do.

And it's fine and workable and good and great, but .. also wrong.

Because my understanding of Unicode is still too simple. 

---

#### There's another layer

I found myself reading a Google preview of [Unicode Demystified: A Practical Programmer's Guide to the Encoding Standard](https://books.google.co.uk/books/about/Unicode_Demystified.html)
by Richard Gillam, and found that you can write `é` as a single thing `[char]0xE9`,
but you can also write it as the letter `e` and the character
`COMBINING ACUTE ACCENT U+0301` next to each other.

I think it may be ["more correct"](http://unicode.org/reports/tr15/#Norm_Forms) to do that. 
It's the "Canonical Decomposition" which allows you to take input text using
both ways of writing `é`, make them into a standard form,
and then compare them to see if they are equal.

    PS C:\> 'é'.Normalize([System.Text.NormalizationForm]::FormD)
    # this splits the "one character" into two, the canonical form
    # but I can't copy and paste them here because they just recombine
    # 🤷

Worse, Mr Gillam writes that there are combining characters which attach to two base characters.
Like [`COMBINING DOUBLE TILDE U+0360`](https://www.fileformat.info/info/unicode/char/0360/index.htm)
which covers the previous and following character. Let's see how that works... `ãa` vs `a͠a`.

And worse again, Mr Gillam says that there can be an arbitrary number of combining character marks.

Now my understanding is more like this:

|          | Classic | Basic Unicode | Next Unicode | Copy of Next Unicode   |
|----------|---------|---------------|--------------|------------------------|
|          |         |               | "ă"          | "ă"                    |
|          |         |               | Unicode?     | Unicode?               |
|          | "a"     | "aé"          | U+0103       | (Multiple CodePoints?) |
|          | ASCII   | Unicode?      | UTF8         | (Choice of Encoding)   |
| byte(s): | 97      | 97 233        | 196 131      | ¿Dragons here🐉        |


But the summary is: there isn't any way to cut the last 20 characters from a file.

Display graphemes (a thing on-screen which looks like a character) might be made
of multiple code points, which might be variable number of byte encoded. 
In the worst case the 2Gb file might be a single base character and two billion combining accents.

If you jump back 20 bytes, or 200 bytes, or 2000 bytes from the end, 
you might still end up in the middle of a multibyte character.
Or the middle of a multi-codepoint-grapheme.

And the only way to /know/ is to read from the beginning.

And even if you read 2Gb of bytes from the beginning,
it doesn't make sense to `.SubString()` it to take "some characters"
out of the end. Because it might split in the middle of a multi-character-grapheme.

Although for English speakers and most files, you can do it,
and it works and it's fine and convenient. 
You just can't do it properly quickly. Or at all.

The preview of the book is quite readable, 
(I doubt I could make it through 800+ pages though),
but it probably contains many other reasons this can't work 100% correctly.

Which means all the things from earlier programming lessons aren't quite right.

- Characters don't map 1-1 to numbers
- or to a fixed set of numbers
- or to a single Unicode codepoint
- strings aren't as simple as multiple characters
- reversing a string isn't easy and might be impossible
- checking if a string is a palindrome..
- oh heck, google "wrong facts programmers believe about Unicode" there's probably thousands of these things.

