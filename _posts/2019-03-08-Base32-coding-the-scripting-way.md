---
layout: post
title: Base32 encoding and decoding, the scripting way
date: 2019-03-08
tags: [PowerShell]
---

For my previous [Google Authenticator post](https://humanequivalentunit.github.io/Google-Authenticator-In-PowerShell/), 
the secret codes involved (e.g. `5WYYADYB5DK2BIOV`) are BASE32 encoded data.
I want to write about how I did that in Powershell,
but you might need to know what an encoding is first.

If you know about encodings, [skip over this next part](#skipped-encoding).

#### How do you pass a blob of data through the internet?

A file upload site, yes,
but wouldn't it be convenient if you could send files through email, forums, blog posts, 
and use *text* processing tools on them?



You can take any file, treat it as a sequence of bytes,
each of those is 8-bits, which is also a number 0-255:

```powershell
PS C:\> get-content c:\windows\system32\calc.exe -Encoding Byte | 
          select -First 3
77
90
144

PS C:\> get-content c:\windows\system32\calc.exe -Encoding Byte | 
            select -First 3 |
            foreach { [convert]::ToString($_, 2).padleft(8, '0') }
01001101
01011010
10010000
```

Roughly speaking, if you split a byte in half, 
each half can be numbers 0-15.
Use "A = 1, B = 2, C = 3" and find the letters for each half:

```powershell
10101101    (one hundred and seventy three)
1010   1101
(ten)  (thirteen)
 J      M
```

Now you have the byte value 173 turned into the text `JM`,
and can send it over any text tool, any forum, any source control
as long as you can undo this, and get the bytes back.
This process is used for things like SSL certificates, 
the stuff inside here is BASE64 encoded:

    -----BEGIN CERTIFICATE-----
    MIIF2zCCBMOgAwIBAgIQMj8HjgweXk
    
The real BASE64 and BASE32 do not split bytes in half,
because they can do better and the output won't be so much bigger.

#### Skipped encoding intro

For BASE32 encoding you take 5-bits at a time,
treat them as numbers 0-31,
and look them up in the character set  `'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567'`. 
For decoding you do this backwards,
character to charset position, to 5-bit value.

This is a right pain, 
look at how 5-bit chunks do not line up nicely with 8-bit bytes:

```
$bytes = [byte[]] @( 77, 90, 144 )
01001101 01011010 10010000
|||||          
     ||| ||
           |||||
                | ||||
                      ...
```

Keeping track of how many bits you have taken from each byte,
doing the bit-manipulation to take some bits out of one byte and out of another and combining them,
is a lot of careful book-keeping.
Proper software developers will care about what happens if there aren't enough.

Take a cutting from [this C# code](https://stackoverflow.com/a/7135008) to turn a byte array to a BASE32 string:

```csharp
public static string ToString(byte[] input)
{
    int charCount = (int)Math.Ceiling(input.Length / 5d) * 8;
    char[] returnArray = new char[charCount];

    byte nextChar = 0, bitsRemaining = 5;
    int arrayIndex = 0;

    foreach (byte b in input)
    {
        nextChar = (byte)(nextChar | (b >> (8 - bitsRemaining)));
        returnArray[arrayIndex++] = ValueToChar(nextChar);

        if (bitsRemaining < 4)
        {
            nextChar = (byte)((b >> (3 - bitsRemaining)) & 31);
            returnArray[arrayIndex++] = ValueToChar(nextChar);
            bitsRemaining += 5;
        }

        bitsRemaining -= 3;
        nextChar = (byte)((b << bitsRemaining) & 31);
    }

    return new string(returnArray);
}
```

Read it and tell me if there are any off-by-one errors or weird edge cases.

No no no, up here in scripting land we have plenty of CPU to waste.
We're going to use strings and regex.

```powershell
[byte[]] $bytes = @(188, 138, 225, 120, 123, 253, 45, 228, 185, 61)

$byteArrayAsBinaryString = -join $bytes.ForEach{
        [Convert]::ToString($_, 2).PadLeft(8, '0')
    }

$Base32Secret = [regex]::Replace($byteArrayAsBinaryString, '.{5}', {
        param($Match)
        'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567'[[Convert]::ToInt32($Match.Value, 2)]
    })

# XSFOC6D37UW6JOJ5
```

The regex `.{5}` picks out five "bits" at a time, 
and there's two calls to `[convert]` to go between numbers and strings-representing-binary.

As long as there are a multiple of 5 bits in the byte array, 
which is fine for my use case. 
It's horrendously inefficient if we were encoding megabytes of data, 
but this is only ten bytes, it's fast enough.

#### Decoding
----

and the C# code to go back the other way,
BASE32 string to byte array is as long and full of bookkeeping:

```c-sharp
public static byte[] ToBytes(string input)
{
       
    input = input.TrimEnd('='); //remove padding characters
    int byteCount = input.Length * 5 / 8; //this must be TRUNCATED
    byte[] returnArray = new byte[byteCount];

    byte curByte = 0, bitsRemaining = 8;
    int mask = 0, arrayIndex = 0;

    foreach (char c in input)
    {
        int cValue = CharToValue(c);

        if (bitsRemaining > 5)
        {
            mask = cValue << (bitsRemaining - 5);
            curByte = (byte)(curByte | mask);
            bitsRemaining -= 5;
        }
        else
        {
            mask = cValue >> (5 - bitsRemaining);
            curByte = (byte)(curByte | mask);
            returnArray[arrayIndex++] = curByte;
            curByte = (byte)(cValue << (3 + bitsRemaining));
            bitsRemaining += 3;
        }
    }

    return returnArray;
}
```

and the PowerShell would be shorter by using BigInteger to make combining 5-bits at a time easier,
but nothing is quite as simple as I hope and there are two catches involved:

```powershell
    $bigInteger = [Numerics.BigInteger]::Zero
    
    foreach ($char in ($secret.ToUpper() -replace '[^A-Z2-7]').GetEnumerator()) {
        $bigInteger = ($bigInteger -shl 5) -bor ('ABCDEFGHIJKLMNOPQRSTUVWXYZ234567'.IndexOf($char))
    }

    [byte[]]$secretAsBytes = $bigInteger.ToByteArray()
    

    # BigInteger sometimes adds a 0 byte to the end,
    # if the positive number could be mistaken as a two's complement negative number.
    # If it happens, we need to remove it.
    if ($secretAsBytes[-1] -eq 0) {
        $secretAsBytes = $secretAsBytes[0..($secretAsBytes.Count - 2)]
    }


    # BigInteger stores bytes in Little-Endian order, 
    # but we need them in Big-Endian order.
    [array]::Reverse($secretAsBytes)
```

The C# could do this as well.

Anyway, the BASE32 regex was quite fun,
and now you know where my comment "I can't write the alphabet" from the previous blog post fits in,
and why it broke my encoding/decoding.
