---
title: "Moving SQL Server System Databases With PowerShell"
description: "Using a PowerShell Script to automatically move both your SQL System Databases and System Log files to dedicated volumes."
pubDate: "2021-05-19T11:00:00.000Z"
heroImage: "https://images.unsplash.com/photo-1555066931-4365d14bab8c?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1170&q=80"
# category slug: choose from "./src/data/category.js"
category: "technology"
# remove this line to publish
draft: false
# author slug: choose from "./src/data/authors.js"
author: "dane-alex"
tags: [SCCM, PowerShell]
---

## Using a PowerShell Script to automatically move both your SQL System Databases and System Log files to dedicated volumes.

One task that I was frequently finding myself spending more time than necessary with was building out SQL Servers. The chief time consuming task being when I'd have to move the SQL Server's default System Database and Log Files to new, dedicated database and log volumes.  

I figured creating a script where I can just run it and have it move the master, model and msdb mdf and ldf files for me would be the best way to resolve this, but wasn't sure if it was something that could reasonably be done. However, after doing some brief research on how to handle this, I came across a great article by Mike Fal. His article covers the basics of using the PowerShell Modules for interacting with SQL, as well as moving the database files to a new location. I expanded on the code snippets in his article, with the script below which allows moving the database and log files to different destinations if desired.

```powershell
### LOAD NECESSARY MODULES ###
##############################

Install-Module SQL-SMO
[System.Reflection.Assembly]::LoadWithPartialName('Microsoft.SqlServer.SqlWmiManagement')

### SET PATH PARAMETERS ###
###########################

$defaultSystemDatabasePath = "E:\Program Files\Microsoft SQL Server\MSSQL14.MSSQLSERVER\MSSQL\DATA"
$desiredSystemDatabasePath = "F:\Databases"
$desiredSystemDatabaseLogPath = "L:\Logs"

### BEGIN MOVING DATABASES ###
##############################

$smo = New-SMO -ServerName localhost -Verbose

# Alter Model and MSDB File Paths in SQL 

$('model','MSDB') |
    ForEach-Object {$Db = $smo.databases[$PSItem]
        foreach ($fg in $Db.FileGroups) 
        {foreach ($fl in $fg.Files) {$fl.FileName = $fl.FileName.Replace($defaultSystemDatabasePath, $desiredSystemDatabasePath)}}
        foreach ($fl in $Db.LogFiles) {$fl.FileName = $fl.FileName.Replace($defaultSystemDatabasePath, $desiredSystemDatabaseLogPath)}
        $smo.databases[$PSItem].Alter()
    }

Stop-Service -Name MSSQLSERVER -Force -Verbose

# Physically Move the Files To The New Directory

$('mast', 'model','MSDB') | ForEach-Object {Move-Item -Path $($defaultSystemDatabasePath + '\' + $PSItem+'*.mdf') -Destination $($desiredSystemDatabasePath + "\") -Verbose}
$('mast', 'model','MSDB') | ForEach-Object {Move-Item -Path $($defaultSystemDatabasePath + '\' + $PSItem+'*.ldf') -Destination $($desiredSystemDatabaseLogPath + "\") -Verbose}

# Update Startup Parameters for Master DB and Log

$wmisvc = $(New-Object Microsoft.SqlServer.Management.Smo.Wmi.ManagedComputer 'localhost').Services | where {$_.name -eq "MSSQLSERVER"}
$wmisvc.StartupParameters= "-d$desiredSystemDatabasePath\master.mdf;-e$desiredSystemDatabasePath\ERRORLOG;-l$desiredSystemDatabaseLogPath\mastlog.ldf"
$wmisvc.Alter()

Start-Service -Name MSSQLSERVER,SQLSERVERAGENT -Verbose
```

The variables are set at the top of the script for the Default System Database Path, or where SQL dropped those databases during the install process and then the Desired Database and Log Paths that you want them to end up at.

The script should then process and tell SQL to update with the new paths the files will be located at, stop the SQL services, move the physical mdf/ldf files, modify the startup parameters for the master database changes and, finally, restart the SQL services.