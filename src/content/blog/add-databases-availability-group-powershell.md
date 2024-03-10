---
title: "Add SQL Databases to an Availability Group with PowerShell"
description: "I recently came across the need to add all databases on a server to a new availability group that was configured. I ended up working on a small PowerShell script that could address this."
pubDate: "2021-06-30T11:39:36.050Z"
heroImage: "https://images.unsplash.com/photo-1555066931-4365d14bab8c?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1170&q=80"
category: "technology"
author: "dane-alex"
tags: [SQL, PowerShell]
---

I recently came across the need to add all databases on a server to a new availability group that was configured. I had a few small problems:

* A lot of the databases were in Simple Recovery mode.
* Some databases were already in the AG.
* I didn't want to have to manually add 20+ databases to the AG manually.
* I needed to ignore the system databases and add a way to ignore dbs that shouldn't be in the AG.

I ended up working on a small PowerShell script that could address these things. It checks for DBs already in the AG, DBs that are in Simple Recovery mode get changed to full, and finally it takes a backup and adds them to the AG. I also created a simple list of database names to use as an ignore list. This will address the issue of system databases and other databases that should not be added to the AG when running the script.

To use the script, you can modify the SET VARIABLES section of the script, which you can identify with the comment blocks.

```powershell
##############################################################################
#.SYNOPSIS
# Add SQL Databases to An Availability Group
# This should be run on the current primary server in your AG. 
#
#.DESCRIPTION
# This script will iterate over all databases on the local server
# and attempt to add them to to a SQL Availability Group (AG) of your choosing.
#
# First, the script will check if the database is not already in an AG,
# or if it is in our "Ignore List."
#
# Second, the script will check to see if the database is in Simple Recovery 
# Mode and change it to Full. 
# 
# Third, initiate a backup of the Full Recovery DB.
#
# Last, it will add the databases to the AG.   
#
#.NOTES
# AUTHOR: DaneAlex 
#
#.EXAMPLE
# First, modify the Set Variables section of this script before running. 
#
# If the server blocks this script, try running the below first:
# Set-ExecutionPolicy -ExecutionPolicy Bypass -Force
#
# .\Add-DatabasesToAG.ps1
##############################################################################

###############################
# Load SQL Modules
###############################
Install-Module SQL-SMO
Import-Module SQL-SMO
Add-Type -AssemblyName "Microsoft.SqlServer.Smo,Version=14.0.0.0,Culture=neutral,PublicKeyToken=89845dcd8080cc91"

###############################
# Set Variables 
###############################
$backupDir = "G:\Backups"
$AGName = "Name_Of_SQL_AG"
$primaryNode = $env:COMPUTERNAME
$databasesToIgnore = "master", "model", "msdb", "tempdb"

###############################
# Begin Script
###############################

# Connect to sql instance and get databases
$sqlinstance = New-Object -TypeName Microsoft.SQLServer.Management.Smo.Server("localhost")
$dbs = $sqlinstance.Databases

# Iterate over all DBs found on this instance of SQL.
foreach($db in $dbs){

    #################################
    # AG and Ignore Checks
    #################################
    
    if($db.AvailabilityGroupName -ne "" -or $databasesToIgnore -contains $db.Name){ 
        Write-Host "$(Get-Date -format g) - INFO - $($db.Name) - Skipping because it is either in the ignore list, or already in an AG."
        continue 
    }

    Write-Host "$(Get-Date -format g) - INFO - $($db.Name) - Is not currently in the AG. "

    #################################
    # Recovery Model Checks
    #################################

    if($db.RecoveryModel -eq "Simple"){

        Write-Host "$(Get-Date -format g) - INFO - $($db.Name) - DB in Simple Recovery. Attempting to switch to Full Recovery model."

        try{
            # Change recovery model if Simple.
            $db.RecoveryModel = "Full";
            $db.Alter();
        }catch{
            # Write an error if change fails, and skip to next database. 
            Write-Host "$(Get-Date -format g) - ERROR - $($db.Name) - Failed to set databases to Full Recovery Model."
            continue 
        }

        Write-Host "$(Get-Date -format g) - INFO - $($db.Name) - Attemping to take a full backup server after recovery model change."
        try{
            # Create a SQL backup object for this database
            $dbBackup = new-object ("Microsoft.SqlServer.Management.Smo.Backup")
            $dbBackup.Database = $db.name
            # Set SQL backup location and type (e.g. Database, Log)
            $dbBackup.Devices.AddDevice("$backupDir\$($db.Name)_AAG_Setup.bak", "File")
            $dbBackup.Action = "Database"
            # Execute the backup
            $dbBackup.SqlBackup($sqlinstance)
        }catch{

            # Write an error if backup fails, and skip to next database. 
            Write-Host "$(Get-Date -format g) - ERROR - $($db.Name) - Failed attemping to take a backup."
            continue 
        }
    }

    #################################
    # Add Database to AG
    #################################

    Write-Host "$(Get-Date -format g) - INFO - $($db.Name) - Attemping to add database to AG on $primaryNode."

    try{
        # Run SQL Query to add current database to the AG.
        $queryPrimary = "ALTER AVAILABILITY GROUP [$($AGName)] ADD DATABASE [$($db.Name)]"
        Write-Host "$(Get-Date -format g) - INFO - $($db.Name) - $($primaryNode):" $queryPrimary
        Invoke-Sqlcmd -Query $queryPrimary
    }catch {
        # Write an error if it failed to add to AG and go to next database.  
        Write-Host "$(Get-Date -format g) - ERROR - $($db.Name) - $($primaryNode): Failed to add to AG"
        continue
    }
    # You can uncomment this exit if you want to do a test against the first db only. 
    # Exit 0

}
```