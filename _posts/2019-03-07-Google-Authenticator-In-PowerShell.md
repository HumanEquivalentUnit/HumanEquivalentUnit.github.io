---
layout: post
title: Google Authenticator in PowerShell
date: 2019-03-07
tags: [PowerShell]
---

```powershell
PS C:\> Import-Module c:\downloads\GoogleAuthenticator.psm1
PS C:\> Get-GoogleAuthenticatorPin -Secret XSFOC6D37UW6JOJ5 | Format-List

PIN Code          : 030 191
Seconds Remaining : 27
```

But where do you get that secret code?

Relevant links:
 - [This blog post's PowerShell code - GoogleAuthenticator.psm1](https://github.com/HumanEquivalentUnit/PowerShell-Misc/blob/master/GoogleAuthenticator.psm1)
 - [An online QR code reader](https://webqr.com/) - click the 'still camera' icon on the top right, then you can drag QR codes to it.
 - [Google Authenticator app on iTunes store](https://itunes.apple.com/gb/app/google-authenticator/id388497605?mt=8) (optional)
 - [Google Authenticator app on Play store](https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2) (optional)

Google Authenticator is a 2-Factor Authentication (2FA) system, 
with an app that generates codes like this:

![Google Authenticator iPhone App Screenshot](/images/2019-03-07-GoogleAuthenticatorApp.png)

I wanted to generate that PIN code in PowerShell.


----

A basic website login has a username and password; 
anyone in the world who steals your password can get into your account.
Google Authenticator 2FA adds another code from a smartphone app,
and now anyone logging in needs to know your password *and* have your smartphone.

Behind the scenes, there is another secret stored against your user account and shared between the server and your smartphone.
It gets there through a QR code, here's an example of a basic logon and enabling 2FA:

![Example of basic website logon form vs combined 2FA signup and logon form](/images/2019-03-07-GoogleAuthenticatorLoginExample.png)

If you have a signup for one of these Google 2FA sites, 
visit the online QR code reader linked above
and see that the QR code contains [text in the Key Uri format](https://github.com/google/google-authenticator/wiki/Key-Uri-Format) like this:

```
otpauth://totp/{accountName}?secret={code}&issuer={companyName}
                                     ^
                                      this is the secret
``` 
    
The first thing this PowerShell module can do is generate one of these random secrets,
wrap it in this `otpauth://` style link,
then embed that in a link to Google Charts to show the whole thing as a QR code.

```powershell
PS C:\> Import-Module c:\temp\GoogleAuthenticator.psm1
PS C:\> $Secret = New-GoogleAuthenticatorSecret -Online    # online will open a browser window
PS C:\> $Secret | Format-List

Secret    : XSFOC6D37UW6JOJ5
QrCodeUri : http://chart.apis.google.com/chart?cht=qr&chs=200x200&chl=otpauth%3A%2F%2Ftotp%2FExample%2520Website%253Aal
            ice%2540example.com%3Fsecret%3DXSFOC6D37UW6JOJ5%26issuer%3DExample%2520Corp
```

You can scan the QR code in the app to add your new token.

Once you have the secret code, 
either generating your own (for fun) or taken from a real website's QR code for your account,
you can generate the current PIN code for it:

```powershell
PS C:\> $Secret | Get-GoogleAuthenticatorPin | Format-List

PIN Code          : 030 191
Seconds Remaining : 27
```

I do not know of a way to get the secret out of the Google Authenticator app, 
but it does have a bit of a weakness - if you are enabling 2FA
you can scan the QR code with as many devices as you want and they will all generate the same PIN codes.

So disable/re-enable, or reset, 2FA on your account (careful not to lock yourself out),
setup your normal login again with a new QR code and add that to the app - but also take a copy.
Now you can get the secret out of the QR code URL,
and generate the PIN from PowerShell.

NB. that this is a lot like having a password in plain text,
and should be treated as carefully - store it somewhere safe, preferably an encrypted password vault, etc.

Playing with this code / the QrCodeUri and Google Charts would also let you
add your serious website secret to the app, but with a more amusing name.
Re-do your 2FA to get a new QR code, and get the `otpauth://` text out of it, and get the secret code.

Generate a new code with your custom username and company name to show up in the app:

```powershell
PS C:\> New-GoogleAuthenticatorSecret -Name 'My Login!' -Issuer 'Some Company' | % QrCodeUri
https://chart.apis.google.com/chart?cht=qr&chs=200x200&chl=otpauth%3A%2F%2Ftotp%2FMy%2520Login%2521%3Fsecret%3D6HHNKDJM5RWHBHJY%26issuer%3DSome%2520Company
```

Then swap the secret part after `secret%3D` with your real code (%3D is = so leave that in),
visit that in a browser, scan into the app - a working login, with custom text.

----

On the PowerShell side of things, I started out with [this StackExchange explanation](https://security.stackexchange.com/a/135953) of the algorithm,
and trying to port [this GoLang code](https://github.com/robbiev/two-factor-auth),
and after getting code which looked OK but generated the wrong output,
I worked from [this C# code](https://stackoverflow.com/questions/6421950/is-there-a-tutorial-on-how-to-implement-google-authenticator-in-net-apps)

The algorithm itself is not very complex, but I did trip over a lot of minor problems:

 - Intel CPUs / dotnet on Windows is little-endian byte order, the algorithm needs big-endian byte order
 - Base32 encoding is involved and that means turning 8-bit-bytes into 5-bit-chunks, and I didn't want to write a whole lot of bit manipulation.
  - BigInteger class helped, but it stores bytes little-endian inside and has occasional padding for handling two's complement negative numbers.
  - After trying some combinations of BitConverter, and starting to port a page of C# code just for byte work, I switched to use regex and "binary" strings instead.
 - `[BitConverter]::GetBytes($time)` was trying to hit the overload for `[char]` until I added a `[int64]` type, and it was sometimes working.
 - I can't type the alphabet correctly and put "VXWYZ", so some codes which included X or W would cause the wrong output.
 - dotNet has a mess of methods for URI Encoding, [8 different ones](https://stackoverflow.com/a/21771206) for different framework versions, which are variously broken, incompatible with RFCs, or non-standard.
  - (I'm not sure I picked a good one)

NB. Google Authenticator app, and the TOTP protocol support more features than I have coded - other window sizes, other hash algorithms.

