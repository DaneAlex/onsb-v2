---
title: "Using Enums in PowerShell"
description: "Some description"
pubDate: "2023-08-23T10:31:36.050Z"
heroImage: "https://images.unsplash.com/photo-1555066931-4365d14bab8c?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1170&q=80"
# category slug: choose from "./src/data/category.js"
category: "technology"
# remove this line to publish
draft: false
# author slug: choose from "./src/data/authors.js"
author: "dane-alex"
tags: [powershell]
---

I have prepped a number of very short, but useful, scripts in PowerShell overtime for my team to utilize in their day to day. 

Most of these are really simple scripts, but can take a number of inputs to do things like manage Sharepoint Sites. 

I was trying to find a better way to sanitize/validate the input that my team may use, to make the scripts easier to follow/use. 

In thinking through my use of enums in Rust, I decided to see if PowerShell had enums available as well. Turns out, it absolutely does in PowerShell versions >= 5.0. [PowerShell Enums](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_enum?view=powershell-7.3)

Below is a basic use case where I set a SharePoint site to readonly or unlocked, and figured the enum was an easy way to show the team the available options, without having to write out comments. 

```powershell
## Site Status Options 
enum SiteStatus {
    ReadOnly
    Unlock
}

# Change to ReadOnly or Unlock after the '::', per the enum above.
$siteStatus = [SiteStatus]::Unlock 
```

The only part I haven't quite figured out, is handling an invalid enum option without having to write a $null check. By default, the enum will present only available options, but a user could still input something incorrect. 

If I were using this in a function that intakes parameters, I could make the SiteStatus enum, one of the parameters and it would automatically fail on invalid input. However, since this is a script, I used the following workaround for now to throw a warning on invalid input:

```powershell
if ($null -eq $siteStatus){
    Write-Warning "Invalid site Status Set. Please set the siteStatus Variable to a valid option from the SiteStatus enum."
    return
    
} 
```

If anyone can think of a cleaner way to handle that in a script based format, please reach out as I'd love to hear your thoughts. 

Either way, I'm happy with it showing the team what options are available and also allowing me to validate invalid input a little bit easier, without them having to read too much into the script itself. 

