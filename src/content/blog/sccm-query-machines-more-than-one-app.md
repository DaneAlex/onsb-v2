---
title: "SCCM Query - PCs with More than One Software"
description: "A quick post on an SCCM Query to find PCs that have more than one piece of software installed. (e.g. Adobe Reader DC and Adobe Reader XI)"
pubDate: "2021-05-21T11:00:00.050Z"
heroImage: "https://images.unsplash.com/photo-1555066931-4365d14bab8c?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1170&q=80"
# category slug: choose from "./src/data/category.js"
category: "technology"
# remove this line to publish
draft: false
# author slug: choose from "./src/data/authors.js"
author: "dane-alex"
tags: [SCCM, PowerShell]
---

Really short post for today! I recently had to identify the easiest way to accurately query machines that had more than one piece of software installed, and make a collection from them.  

I did a bit of searching online, and a lot of the responses I found didn't actually work because of the way the query was structured.

I went into the Config Manager database using SSMS and played with some of the Queries there, and finally wound up with the one below that seemed to return in the most efficient time:

```sql
select
    SMS_R_SYSTEM.ResourceID,
    SMS_R_SYSTEM.ResourceType,
    SMS_R_SYSTEM.Name,
    SMS_R_SYSTEM.SMSUniqueIdentifier,
    SMS_R_SYSTEM.ResourceDomainORWorkgroup,
    SMS_R_SYSTEM.Client 
from
    SMS_R_System       
where
    SMS_R_System.Name in (
        select
            SMS_R_System.Name               
        from
            SMS_R_System               
        inner join
            SMS_G_System_INSTALLED_SOFTWARE                       
                on SMS_G_System_INSTALLED_SOFTWARE.ResourceID = SMS_R_System.ResourceId               
        where
            SMS_G_System_INSTALLED_SOFTWARE.ARPDisplayName like "%Application1%"          
    )           
    and SMS_R_System.Name in (
        select
            SMS_R_System.Name               
        from
            SMS_R_System               
        inner join
            SMS_G_System_INSTALLED_SOFTWARE                       
                on SMS_G_System_INSTALLED_SOFTWARE.ResourceID = SMS_R_System.ResourceId               
        where
            SMS_G_System_INSTALLED_SOFTWARE.ARPDisplayName like "%Application2%"          
    )
```

Other queries seemed to be trying to hit the Add or Remove Programs and Add or Remove Programs 64 tables which mean double the searches for both applications.