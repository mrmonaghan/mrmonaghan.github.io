---
layout: post
title: Monitoring Windows Server Backups
---

Windows Server Backup is a lot like my first car - it does almost nothing that I wish it did, but it gets you where you need to go. Thankfully, we can tune it up a bit with some aftermarket Powershell to make WSB a more respectable disaster recovery whip.

One thing that I like to know about my backups is when they are or are not working. Unforunately, WSB requires us to manually review it's logs to determine if backups are completing successfully. As part of my ongoing initiative to do as little as possible on a regular basis, I've developed a quick script to keep me in the loop.

The first thing we need to do is identify what happens when a WSB backup job fails - if we take a look at the Event Logs under Microsoft-Windows-Backups, we can see that the Operational log contains all of the information we need to power our notifications. There are lots of ways to review these logs from the command line like [Get-WinEvent](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent?view=powershell-6) and [Get-EventLog](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-eventlog?view=powershell-5.1). I was recently recommended a method of filtering Get-WinEvent by [u/OlivTheFrog](https://www.reddit.com/user/OlivTheFrog) over on the Powershell subreddit that I have been having a lot of success with, and that I will employ here. It utilizes the -FilterHashtable parameter to markedly increase the speed at which your script queries events in additon to providing a more readable filter as you can see below:

{% highlight powershell %}
{% raw %}
#Define ProviderName
$ProviderName = "Microsoft-Windows-Backups"

#Build lists of events based on $ProviderName and Event Level
$Errors = Get-WinEvent -FilterHashtable @{ProviderName=$ProviderName
                                          Level='1','2','3'
                                          }
$Information = Get-WinEvent -FilterHashtable @{ProviderName=$ProviderName
                                               Level='4'
                                               }
{% endraw %}
{% endhighlight %}

As you can see, the syntax is very simple. Rather than using Where-Object to filter an entire list of objects, -FilterHashtable allows us to skip the garbage and go directly to the objects that match our filter. I use the Level property to filter events based on their severity, and save them into seperate variables for later.

Next we need to figure out what's relevant. Unlike other backup softwares, we can't clear the Error Event once we've resolved the issue (well, we can, but that kind've defeats the purpose of the Event Log). Instead, we can filter by the TimeCreated property to find out what is happening recently.

{% highlight powershell %}
{% raw %}
$RecentErrors = $Errors | Where-Object {$_.TimeCreated -ge (Get-Date).AddDays(-1)}
$RecentSuccess = $Information | Where-Object {$_.TimeCreated -ge (Get-Date).AddDays(-1)}
{% endraw %}
{% endhighlight %}

We can't include our TimeCreated filtering in the hashtable for two reasons:
  1. -FilterHashtable doesn't support the TimeCreated property, only StartTime and EndTime
  2. Because we're comparing two [datetime] objects to determine if something happened within the last 24 hours, StartTime and EndTime won't do us much good. 
  
The above code will build two lists, one of Information level events and one that contains Critical, Error, and Warning level events. Both $Recent lists will only contain events that occured within the last 24 hours. We'll be running this script as a scheduled task as opposed to triggering it on event occurance because we don't necessarily know what specific error, will occur and presumably we want to be notified of any issue with our precious backups.

Now that we have our errors nicely packed into an array, we can build our notification.

Anyone that's tried to sent notifications with Powershell knows two things: 
  1. It's surprisingly easy
  2. Formatting Powershell objects into a readable format is less easy

To remedy this, we're going to have to slum it a little and build some HTML.

{% highlight powershell %}
{% raw %}
#Define mailing information
$To = "recipient@mail.com"
$From = "sender@mail.com"
$SMTPServer = "smtp.webaddress.com"
#Build Email Formatting
$HTMLBody = @()
$Header = @"
            <style>
            body { background-color:#FFFFFF;
            font-family:Tahoma;
            font-size:12pt; }
            h3, h4 {text-align:center;}
            td, th { border:1px solid black;
            border-collapse:collapse; }
            th { color:white;
            background-color:black; }
            table, tr, td, th { padding: 2px; margin: 0px }
            table { width:95%;margin-left:5px; margin-bottom:20px;}
            </style>
"@
$HTMLBody += $Header
{% endraw %}
{% endhighlight %}

I know, I know - but it's worth it. The above code defines a simple black-and-white table with a title and some nice column headers based on the property names of the objects we're going to pipe into it. I am by no means a master of HTML, so it's very possible that someone more knowledgeable could improve on this snippet.

Next, we need to determine what we're going to populate our table with. We do this by taking a look at $RecentErrors:

{% highlight powershell %}
{% raw %}
#If $RecentErrors contains data, build it into the $HTMLBody table and sent the notification. Else, build $RecentSuccesses into $HTMLBody and send the ntoification
if ($RecentErrors.count -gt 0) {
    $HTMLBody += '<h3>The following backup errors occurred:</h3>'
    $HTMLBody += $RecentErrors | Select-Object TimeCreated,Id,LevelDisplayName,Message | ConvertTo-Html -Fragment | Out-String
    Send-MailMessage -To $To -From $From -SmtpServer $SMTPServer -Subject "Backup Errors" -BodyasHtml -Body "$HTMLBody"
    }
else {
    $HTMLBody += '<h3>Backups succeeded! See related Event Logs below:</h3>'
    $HTMLBody += $RecentSuccess | Select-Object TimeCreated,Id,LevelDisplayName,Message | ConvertTo-Html -Fragment | Out-String
    Send-MailMessage -To $To -From $From -SmtpServer $SMTPServer -Subject "Backup Success!" -BodyasHtml -Body "$HTMLBody"
    }
{% endraw %}
{% endhighlight %}

I want to pause and discuss these snippets:

{% highlight powershell %}
{% raw %}
$HTMLBody += $RecentErrors | Select-Object TimeCreated,Id,LevelDisplayName,Message | ConvertTo-Html -Fragment | Out-String
$HTMLBody += $RecentSuccess | Select-Object TimeCreated,Id,LevelDisplayName,Message | ConvertTo-Html -Fragment | Out-String
{% endraw %}
{% endhighlight %}

If we were to simply convert these variables into strings and put them in the body of our email, they wouldn't really further our respectable car fantasy - the headers would be all over the place, and long strings would be trunctated down to what you would see in the console. Instead, we convert the objects to HTML (making sure to select only the properties we want in our table, as ConvertTo-HTML will convert all of an object's properties if you let it) and *then* convert that HTML to a string to put into the body of our email. By specifying the -BodyAsHTML parameter on Send-MailMessage, we tell the cmdlet to interpret the HTML string as proper HTML and deliver the message with our sexy new formatting. Take a look:

![_config.yml]({{ site.baseurl }}/images/blogimages/EmailHTML.PNG)

Now, back to the code. As you can see above, if we count the objects contained in $RecentErrors (which, remember, contains Critical, Error, and Warning level events) and it's got something in it, we know there is an issue with the backups. We can then proceed to customize both our $HTMLBody with a table header that indicates we have a problem, and our Send-MailMessage cmdlet with an appropriate subject line. If $RecentErrors does not contain any data, we simply move on and send an email celebrating our backup success that includes the Information level events that occurred within the last day.

And that's pretty much it! A simple notifications script that will make using Windows Server Backup just a little bit more palatable. The full sanitized script is available [at it's dedicated repository here.](https://github.com/mrmonaghan/WindowsBackupNotifications)

Happy scripting!
