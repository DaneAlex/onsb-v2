---
title: "Enabling Microsoft 365 Encryption via Azure Information Protection"
description: "How I resolved issues with being unable to access my encrypted email features in Microsoft 365, following the move to Azure Rights Management and Purview."
pubDate: "2024-03-08T11:00:00.000Z"
heroImage: "https://images.unsplash.com/photo-1555066931-4365d14bab8c?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1170&q=80"
# category slug: choose from "./src/data/category.js"
category: "technology"
draft: false
# author slug: choose from "./src/data/authors.js"
author: "dane-alex"
tags: [Microsoft, "365", Security, Email, Encryption]
---

# Enabling Microsoft 365 Encryption via AIP

Recently, I upgraded my personal 365 account to a Microsoft 365 Business Premium license and I was excited to utilize the Microsoft 365 email encryption features. 

However, I quickly realized that the "Encrypt" button was missing in both Outlook and Outlook Web Access. After some further investigation into my Admin Portal in 365, I found I was also unable to activate any encryption policies in the mail flow portion of the Exchange Admin Center. 

Before you go too far into this blog post, please review the licenses you have and validate they have Azure Information Protection included. 

At the time of writing this post, the three most common licenses I see at clients that include these features are:

1. Microsoft 365 Business Premium
2. Microsoft Enterprise/Office E3
3. Microsoft Enterprise/Office E5

There are of course other licenses and specific licenses for AIP, but again, these are just the three I see purchased most often.

## The Source of My Issue

Microsoft began a migration over to Purview and Azure Rights Management for the encryption features, starting back in 2018. As I have had my personal tenant for much longer than that I, unfortunately, did not have some required features automatically enabled. 

## The Resolution

The quickest way I was able to resolve the issue and enable the required features in my tenant, was by running the below cmdlets in PowerShell with a global admin account:

Install the Azure Information Protection and Exchange Online Management PowerShell Modules if you do not have them:

`Install-Module -Name AIPService`

`Install-Module -Name ExchangeOnlineManagement`

### AIP Service:
```powershell
Connect-AIPService

# Enable the Azure Information Protection Service and Install RMS Templates
Enable-AIPService
```

### Exchange Online
```powershell
Connect-ExchangeOnline

# Enable Automatic Service Updates
Set-IRMConfiguration -AutomaticServiceUpdateEnabled $true

# Enable "Encrypt" button in OWA and Outlook 365
Set-IRMConfiguration -SimplifiedClientAccessEnabled $true

# Enable Licensing 
Set-IRMConfiguration -InternalLicensingEnabled $true
Set-IRMConfiguration -AzureRMSLicensingEnabled $true
```

### Mail Flow Rule(s)

It may take 10 minutes or so for the changes to take effect. Once they have, you will want to configure a Mail Flow Rule that can enable the encryption features for you. 

I did this in my environment by going to the [Exchange Online Admin Center](https://admin.exchange.microsoft.com/#/), and selecting "Mail Flow" -> "Rules."

From there, I hit the "Add a Rule" button, and chose the new "Apply Office 365 Message Encryption and rights protection to messages."

Configure the rule for what is best in your environment. In my case, I chose to encrypt the email if the subject contains the words "Secure" or "Encrypt."

I used the "Encrypt" template as the Rights Management template under the "Rights protect messages with" option. 

Shortly after I completed the above changes, I then had the "Encrypt" button appear in my Outlook Web Access, allowing me to encrypt the outbound email. I sent a test to another address of mine to confirm it was now working as intended. 


 

