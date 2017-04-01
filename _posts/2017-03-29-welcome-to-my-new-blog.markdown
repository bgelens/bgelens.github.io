---
title:  "Welcome to my new blog!"
date:   2017-03-29 15:04:23
---
I've been blogging for while but never really on a blog of my own. I though it would be a good time to change this and figured since everyone is so happy about GitHub pages, why not start looking into this. I'll start gattering all my old blog posts here and after that will start writing new content.

Expect a lot of PowerShell, DSC, Azure, Azure Stack, my first steps in the dev world, tinkering with all sorts of DevOps / CM tooling, etc. It's a good time to be in IT!

```powershell
$Contact = [ordered]@{
    Name      = 'Ben Gelens'
    Job       = 'Consultant'
    MVP       = 'CDM (PowerShell)'
    Company   = 'InSpark'
    EMail     = 'ben@bgelens.nl'
    Twitter   = '@bgelens'
    GitHub    = 'https://github.com/bgelens/'
}
New-Object -TypeName PSObject -ArgumentList $Contact 
```