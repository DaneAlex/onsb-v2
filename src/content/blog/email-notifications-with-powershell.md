---
title: "PowerShell Basics: Add Email Notifications To Our Monitor - Part II"
description: "Creating HTML-based email templates for powershell notifications."
pubDate: "2020-08-27T11:00:00.000Z"
heroImage: "https://images.unsplash.com/photo-1555066931-4365d14bab8c?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1170&q=80"
# category slug: choose from "./src/data/category.js"
category: "technology"
# remove this line to publish
draft: false
# author slug: choose from "./src/data/authors.js"
author: "dane-alex"
tags: [powershell, html, email]
---

Recap
In the first part of this series, we ran through setting up a very basic scheduled task to monitor an application process and service. The script was left with a bit to be desired as it is only logging the data it gathers to a text file. While this is good for a historical view, and potential root cause detection, it doesn't provide a way for us to be actively notified on errors. We will fix this today by adding an email alert to our script that will email us when failures are detected.

To do this, we will be using the Send-MailMessage cmdlet and the Microsoft Office 365 service. This should work with any email relay you have access to.

Storing the Email Account Password
Before we jump straight into the email settings, we have something important to do first. Since we will need to use a password for accessing the Microsoft 365 account, let's convert our password and store it using an encrypted string inside a text file. Open a separate PowerShell window and navigate to where we are running the script, in this case "C:\Scripts\Monitoring". We will then use the Get-Credential cmdlet to be prompted to enter the e-mail account username and password that we want to use when sending the email.

```powershell
cd "C:\Scripts\Monitoring"
$credential = Get-Credential
$credential.Password | ConvertFrom-SecureString | Set-Content encryptedPass.txt
```

As an extra layer of precaution, I usually create a service mailbox that is only used for sending emails in my account and will not store any confidential incoming email. While this is not required, I highly recommend it if you have an available license/mailbox. Something like "alerts@yourdomain.com" works great!

Adding the Email Alert
Now we can begin configuring our email settings, using the encrypted text file we created. We can add an Email Settings list of variables to the top of our script, below our $scriptTitle and $scriptFolderPath variables. I will be showing the email settings I am using, but remember that yours may be different depending on which email service you are using.

We will also want to add an $errorCount variable and set it to 0. This will help us keep track of whether or not an error is detected as well as how many errors are detected.
```powershell
$scriptFolderPath = "C:\Scripts\Monitoring"
$scriptTitle = "SteamMonitor"
$errorCount = 0

#EMAIL SETTINGS
$smtp = "smtp.office365.com"
$smtpPort = 587
$username = "EMAIL ACCOUNT YOU SAVED THE PASSWORD FOR"
$encrypted = Get-Content encryptedPass.txt | ConvertTo-SecureString
$credentials = New-Object System.Management.Automation.PSCredential(
    $username, 
    $encrypted
)
$from = "EMAIL ADDRESS FOR THE ACCOUNT YOU SAVED THE PASSWORD FOR"
$to = "THE RECIPIENT EMAIL ADDRESS"
```

Next, we will scroll down our script to where we were checking if our service and our application are running. In the code sections that contain the error message and writeLog function for the error message, we want to add new lines to add 1 to our error count variable. $errorCount = $errorCount + 1

This takes whatever the errorCount variable currently contains and adds 1 to it.

See below for an example of what our full code should now look like:

```powershell
$scriptFolderPath = "C:\Scripts\Monitoring"
$scriptTitle = "SteamMonitor"
$errorCount = 0

#EMAIL SETTINGS
$smtp = "smtp.office365.com"
$smtpPort = 587
$username = "EMAIL ACCOUNT YOU SAVED THE PASSWORD FOR"
$encrypted = Get-Content encryptedPass.txt | ConvertTo-SecureString
$credentials = New-Object System.Management.Automation.PSCredential(
    $username, 
    $encrypted
)
$from = "EMAIL ADDRESS FOR THE ACCOUNT YOU SAVED THE PASSWORD FOR"
$to = "THE RECIPIENT EMAIL ADDRESS"

function writeLog
{
    [CmdletBinding()]
	Param
    (
    	[Parameter(Mandatory=$true, ValueFromPipelineByPropertyName=$true)]
        [ValidateNotNullOrEmpty()]
        [string]$logMessage,
        
        [Parameter(Mandatory=$false)]
        [ValidateSet("ERROR", "INFO")]
        [string]$state="INFO"
    )
    
    $logTime = Get-Date -Format "MM-dd-yyyy HH:mm"
    $currentDate = Get-Date -Format "MM-dd-yyyy"
    
    switch ($state) {
    	'ERROR' {
        	$logState = "ERROR:"
		}
        'INFO' {
        	$logState = "INFO:"
		}
    }
    
    "$logTime $logState $logMessage" |
	Out-File -FilePath "$scriptFolderPath\$scriptTitle_$currentDate.log" -Append
}

writeLog -state "INFO" -logMessage "==== $scriptTitle has started ===="

$steamService = "Steam Client Service"
$steamServiceCheck = Get-Service $steamService

writeLog -state "INFO" -logMessage "Checking $steamService"

if($steamServiceCheck.Status -eq "Running"){
	$steamServiceMessage = "$steamService is running as expected."
	writeLog -state "INFO" -logMessage $steamServiceMessage
}else{
	$steamServiceMessage = "$steamService is not running!"
    writeLog -state "ERROR" -logMessage $steamServiceMessage
    $errorCount = $errorCount + 1;
}

$steamApplication = "steam"
$steamApplicationCheck = Get-Process $steamApplication -ErrorAction SilentlyContinue | 
Select Name, Description, Responding

writeLog -state "INFO" -logMessage "Checking $steamApplication"

if(!$steamApplicationCheck){

    $steamApplicationMessage = "$steamApplication is not running!"
	writeLog -state "ERROR" -logMessage $steamApplicationMessage
    $errorCount = $errorCount + 1;

}else{

    if($steamApplicationCheck.Responding -eq "True"){
    	$steamApplicationMessage = "$steamApplication is running."
	    writeLog -state "INFO" -logMessage $steamApplicationMessage
    }else{
        $steamApplicationMessage = "$steamApplication is not responding!"
	    writeLog -state "ERROR" -logMessage $steamApplicationMessage
    }

} 
```

Finally, now that we have a way to identify if our script found errors, we can proceed with actually sending an email when there is errors. At the bottom of our script, we will add a check to see if our error count is greater than 0 and if so, send an email:

```powershell
if($errorCount -gt 0){

    $emailSubject = "$scriptTitle - Alert"
    
    $emailBody = "
    	<h3>$errorCount Issues Found</h3>
        <p>$steamApplicationMessage</p>
        <p>$steamServiceMessage</p>
    "
    $sendMailParameters = @{
        From = $from
        To = $to
        Subject = $emailSubject
        Body = $emailBody
        SMTPServer = $smtp
        Port = $smtpPort
        Credential = $credentials
        BodyAsHTML = $True
        UseSSL = $True
    }
    
    Send-MailMessage @sendMailParameters

}
```
Once that is in place, you should be able to stop your test process or service and run the script. If we are setup correctly, you will receive an email that looks similar to below:

![Simple email example](https://storageaccountonsbp8567.blob.core.windows.net/public/email_basic.png)

Now we have basic alerting that will fire an email as frequently as you have your scheduled task from Part I setup.

If you've gotten stuck somewhere along the way, don't forget to join the Discord server and ask for help!

# Improving the Email
While the above email example provides enough information to get exactly what you need, some companies have template standards that need to be followed, etc. I will show a example of how we can convert an existing HTML email template into something we can use with PowerShell to send a better email.

I've taken a very basic HTML email template (because trying to write HTML for emails just sucks...) and added "handlebars" style tags inside the HTML to identify areas we will replace using PowerShell.

```html
<html>
    <head>
        <title>{{EMAIL SUBJECT}}</title>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <meta http-equiv="X-UA-Compatible" content="IE=edge" />
        <style type="text/css">
            * {
                margin: 0;
                padding: 0;
                box-sizing: border-box;
            }

            body,
            table,
            td,
            a {
                -webkit-text-size-adjust: 100%;
                -ms-text-size-adjust: 100%;
            }

            table,
            td {
                mso-table-lspace: 0pt;
                mso-table-rspace: 0pt;
            }

            img {
                -ms-interpolation-mode: bicubic;
            }

            img {
                border: 0;
                height: auto;
                line-height: 100%;
                outline: none;
                text-decoration: none;
            }

            table {
                border-collapse: collapse !important;
            }

            body {
                height: 100% !important;
                width: 100% !important;
                background-color: #b6b6b6;
            }

            .body-content{
                padding-left: 10px;
                padding-right: 10px;
                background-color: #FFFFFF;
            }

            @media screen and (min-width:600px) {
                h1 {
                    font-size: 32px !important;
                    line-height: 32px !important;
                }

                .intro {
                    font-size: 24px !important;
                    line-height: 24px !important;
                }
            }
        </style>
    </head>
    <body>
        <div style="background-color: #ffffff; 
color: #2b2b2b; 
font-family: 'Avenir Next','Segoe UI', Roboto, Helvetica, Arial, sans-serif; 
font-size: 18px; 
font-weight: 400; 
line-height: 28px; 
margin: 0 auto; 
max-width: 600px; 
padding: 0px 0px 20px 0px;">

            <header>
                <div style="background-color:#000000;padding:1em;width:100%">
                    <center><img src="{{EMAIL LOGO}}" alt="" width="160"></center>
                </div>
            </header>

            <main>
                <div class="body-content">
                    <h1 style="color: #000000; font-size: 32px; font-weight: 800; line-height: 32px; margin: 24px 0;">
                        {{EMAIL HEADER}}
                    </h1>
                    <p class="intro" style="color: #000000; font-size: 18px; font-weight: 600; line-height: 28px;">
                        {{EMAIL INTRO}}
                    </p>
                    {{EMAIL CONTENT}}
                </div>
            </main>
        </div>
    </body>

</html>
```

![Email Template Image](https://storageaccountonsbp8567.blob.core.windows.net/public/email_template.png)

Again, this is just a really simplified example and you can do the same thing with any HTML email template you already have, just as long as you know where you want your content from PowerShell to show up. Save the HTML file somewhere you can get access to it with PowerShell. Either the same folder as the script itself, a network share, etc. I've saved mine in the same folder as my script with the name "emailTemplate.html"

This is where things will get fun with our script. Our script is going to read the contents of the HTML template and convert it to a string using the Get-Content cmdlet. It will then replace each of the handlebar placeholders we created earlier with whatever we want using the str.Replace() method.

```powershell
if($errorCount -gt 0){

    $pathToEmailLogo = "URL PATH TO YOUR LOGO"
    $emailSubject = "$scriptTitle - Alert"
    $emailStatus = "$errorCount Issues Found"

    $emailTemplate = Get-Content -Path emailTemplate.html -Raw
    $emailTemplate = $emailTemplate.Replace("{{EMAIL SUBJECT}}", $emailSubject)
    $emailTemplate = $emailTemplate.Replace("{{EMAIL LOGO}}", $pathToEmailLogo)
    $emailTemplate = $emailTemplate.Replace("{{EMAIL HEADER}}", $emailSubject)
    $emailTemplate = $emailTemplate.Replace("{{EMAIL INTRO}}", $emailStatus)
    
    $emailBody = "
        <p>$steamApplicationMessage</p>
        <p>$steamServiceMessage</p>
    "
	$emailTemplate = $emailTemplate.Replace("{{EMAIL CONTENT}}", $emailBody)

    $sendMailParameters = @{
        From = $from
        To = $to
        Subject = $emailSubject
        Body = $emailTemplate
        SMTPServer = $smtp
        Port = $smtpPort
        Credential = $credentials
        BodyAsHTML = $True
        UseSSL = $True
    }
    
    Send-MailMessage @sendMailParameters

}
```

Our email output will now look something like this:

![Final email example](https://storageaccountonsbp8567.blob.core.windows.net/public/email_with_template.png)

Here is the final version of the script:

```powershell
$scriptFolderPath = "C:\Scripts\Monitoring"
$scriptTitle = "SteamMonitor"
$errorCount = 0

#EMAIL SETTINGS
$smtp = "smtp.office365.com"
$smtpPort = 587
$username = "EMAIL ACCOUNT YOU SAVED THE PASSWORD FOR"
$encrypted = Get-Content encryptedPass.txt | ConvertTo-SecureString
$credentials = New-Object System.Management.Automation.PSCredential(
    $username, 
    $encrypted
)
$from = "EMAIL ADDRESS FOR THE ACCOUNT YOU SAVED THE PASSWORD FOR"
$to = "THE RECIPIENT EMAIL ADDRESS"

function writeLog
{
    [CmdletBinding()]
	Param
    (
    	[Parameter(Mandatory=$true, ValueFromPipelineByPropertyName=$true)]
        [ValidateNotNullOrEmpty()]
        [string]$logMessage,
        
        [Parameter(Mandatory=$false)]
        [ValidateSet("ERROR", "INFO")]
        [string]$state="INFO"
    )
    
    $logTime = Get-Date -Format "MM-dd-yyyy HH:mm"
    $currentDate = Get-Date -Format "MM-dd-yyyy"
    
    switch ($state) {
    	'ERROR' {
        	$logState = "ERROR:"
		}
        'INFO' {
        	$logState = "INFO:"
		}
    }
    
    "$logTime $logState $logMessage" |
	Out-File -FilePath "$scriptFolderPath\$scriptTitle_$currentDate.log" -Append
}

writeLog -state "INFO" -logMessage "==== $scriptTitle has started ===="

$steamService = "Steam Client Service"
$steamServiceCheck = Get-Service $steamService

writeLog -state "INFO" -logMessage "Checking $steamService"

if($steamServiceCheck.Status -eq "Running"){
	$steamServiceMessage = "$steamService is running as expected."
	writeLog -state "INFO" -logMessage $steamServiceMessage
}else{
	$steamServiceMessage = "$steamService is not running!"
    writeLog -state "ERROR" -logMessage $steamServiceMessage
    $errorCount = $errorCount + 1;
}

$steamApplication = "steam"
$steamApplicationCheck = Get-Process $steamApplication -ErrorAction SilentlyContinue | 
Select Name, Description, Responding 

writeLog -state "INFO" -logMessage "Checking $steamApplication"

if(!$steamApplicationCheck){

    $steamApplicationMessage = "$steamApplication is not running!"
	writeLog -state "ERROR" -logMessage $steamApplicationMessage
    $errorCount = $errorCount + 1;

}else{

    if($steamApplicationCheck.Responding -eq "True"){
    	$steamApplicationMessage = "$steamApplication is running."
	    writeLog -state "INFO" -logMessage $steamApplicationMessage
    }else{
        $steamApplicationMessage = "$steamApplication is not responding!"
	    writeLog -state "ERROR" -logMessage $steamApplicationMessage
    }

} 

if($errorCount -gt 0){

    $pathToEmailLogo = "URL PATH TO YOUR LOGO"
    $emailSubject = "$scriptTitle - Alert"
    $emailStatus = "$errorCount Issues Found"

    $emailTemplate = Get-Content -Path emailTemplate.html -Raw
    $emailTemplate = $emailTemplate.Replace("{{EMAIL SUBJECT}}", $emailSubject)
    $emailTemplate = $emailTemplate.Replace("{{EMAIL LOGO}}", $pathToEmailLogo)
    $emailTemplate = $emailTemplate.Replace("{{EMAIL HEADER}}", $emailSubject)
    $emailTemplate = $emailTemplate.Replace("{{EMAIL INTRO}}", $emailStatus)
    
    $emailBody = "
        <p>$steamApplicationMessage</p>
        <p>$steamServiceMessage</p>
    "
    $emailTemplate = $emailTemplate.Replace("{{EMAIL CONTENT}}", $emailBody)

    $sendMailParameters = @{
        From = $from
        To = $to
        Subject = $emailSubject
        Body = $emailTemplate
        SMTPServer = $smtp
        Port = $smtpPort
        Credential = $credentials
        BodyAsHTML = $True
        UseSSL = $True
    }
    
    Send-MailMessage @sendMailParameters

}
```