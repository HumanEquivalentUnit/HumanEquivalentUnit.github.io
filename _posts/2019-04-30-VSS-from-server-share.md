---
layout: post
title: PowerShell - access Volume Shadow Copy (Previous Versions) of a network share
date: 2019-04-30
tags: [PowerShell,Other]
---

Jordan Borean has come up with a way to get to 
"Previous Versions" of network shares using Powershell!

[Here is the relevant Reddit comment](https://www.reddit.com/r/PowerShell/comments/b2wenl/using_deviceiocontrol_and_fsctl_srv_enumerate/eiy0j03/),
and it links to [their module code on GitHub](https://gist.github.com/jborean93/f60da33b08f8e1d5e0ef545b0a4698a0)

Example usage:

```powershell
# Annoying issue with older .NET versions, you need to explicitly enable TLS 1.2 for TLS 1.2 only sites
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Invoke-WebRequest -Uri "https://gist.githubusercontent.com/jborean93/f60da33b08f8e1d5e0ef545b0a4698a0/raw/7657ad8e24c595db8ad563cc6dbc434c2bbc01f6/Get-SnapshotPath.psm1" -OutFile Get-SnapshotPath.psm1
Import-Module -Name .\Get-SnapshotPath.psm1
Get-SnapshotPath -Path "\\dc01\c$"
Get-SnapshotPath -Path "\\dc01\c$\Windows"
```

If you use Powershell alone,
it won't find any previous version files, you can't get to them. 
You need to use Windows Explorer first so it can send the right control codes to the server.
This code works out what Explorer is sending to the server and mimics that.
That's why it's cool!


You can then use the `\\SERVER\SHARE\@GMT-XXXX.XX.XX-XX.XX.XX` format of VSS file paths,
and use `Copy-Item` to recover files, etc.

[Link to copy of source, in case github Gist is removed](static-files/2019-04-30-Get-SnapshotPath.psm1.txt)

