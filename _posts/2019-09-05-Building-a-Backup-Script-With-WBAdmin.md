---
layout: post
title: Building a Better (Than Nothing) Backup Solution with Powershell and WBAdmin
---

Sometimes things need to get done, and correct tool for the job is not available. In these situations, it is up to us as seasoned professionals to Improvise, Adapt, and Overcome - like Tom Hanks extracting a rotten tooth with a figure skate.

Recently I was tasked with identifying a backup solution with the following criteria:
  1. Perform regular, full image backups of a machine
  2. Utilize only previously-existing local USB storage
  3. Be as resilient as possible to ransomware
  4. Make use of no external database dependencies like SQL Express
  5. Function on a non-server OS
  6. Cost no money

Some of these restrictions were imposed by the client, some by their LOB software vendor, and some by the environment itself; what resulted was the business-case version of a table flip - a very polite and professional "Screw it, I'll just build the damn thing myself."

Let's start by breaking down and solving the relevant requirements listed above.

### 1. Perform regular, full image backups of a machine
Our first task is to determine what native Windows 10 backup functionality best suites our needs. There are two: File History, and Backup and Restore (AKA Windows Backup, AAKA WBAdmin). For our purposes, File History is no good - we need a full, restorable image of the machine. This leaves WBAdmin, which does have the capability to create a full system image backup.

The WBAdmin command line utility and [its accompanying documentation](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/wbadmin) provide all the information we need to start and monitor a backup job from CMD - but how can we leverage Powershell to integrate that into our solution more elegantly?

The answer resides in Powershells Start-Process and Wait-Process cmdlets, which allow us to launch WBAdmin, assign the process a variable, and wait for the process to exit before continuing.

{% highlight powershell %}
{% raw %}
$Process = Start-Process -File "wbadmin" -ArgumentList 'start backup -backupTarget:E: -include:C: -quiet -allCritical' -PassThru
Wait-Process -InputObject $Process
{% endraw %}
{% endhighlight %}

This ends up being fairly straightforward and readable. We use Start-Process to start the wbadmin application using the -File parameter and supply ALL of the remaining arguments via the -ArgumentList parameter. These all get compiled into a long string, as that is the only way WBAdmin accepts the input. Finally, we include the -Passthru parameter which returns a process object (similar to what would be returned by Get-Process) for each process the Start-Process cmdlet starts. This process object gets assigned to the $Process variable, which we call in the next line.

Now that we've started the process, we need to use Wait-Process to tell the script not to do anything else until the WBAdmin process has completed and exited. Wait-Process accepts a couple of different types of input, but since $Process contains a process object thanks to the -Passthru parameter, we'll supply $Process to Wait-Process via the -InputObject parameter. Now our script will monitor the status of the WBAdmin process, proceeding to the next step only when the process exits.

### 2 & 3. Local USB Storage that's ransomware-resistant
Next, we need to address points 2 and 3 together. Traditionally (and in an ideal scenario) local USB drives make terrible backup targets, because they need to be mounted in order for the backup solution to write files to them, and a mounted drive makes a juicy target for ransomware. Products like Veeam provide a solution for this - automatically mount the backup repsitory before the backup job begins, and unmount it when it completes. This means the drive is only mounted and accessible during the backup window. We may not be Veeam, but we should be able to achieve a similar result [using the mountvol utility.](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/mountvol)

{% highlight powershell %}
{% raw %}
$VolumeID = '<Insert Volume GUID string>'
mountvol E: $VolumeID
{% endraw %}
{% endhighlight %}

Again, the syntax here is pretty simple. First, we need to define the $VolumeID variable using the DeviceID property of our local USB drive. Unforunately, we aren't able to query this information while the drive is unmounted, as it is the only identifer provided while the drive is in an unmounted state. To get the DeviceID of a mounted drive, run the following command:

{% highlight powershell %}
{% raw %}
Get-WmiObject -Class Win32_Volume | Where-Object {$_.label -like "<ENTER BACKUP DRIVE NAME OR PARTIAL NAME HERE>"} | Select-Object -ExpandProperty DeviceID
{% endraw %}
{% endhighlight %}

Which will return a string that looks like this \\?\Volume{0000e0e0-0000-0000-0000-000000000000}\ which will become your $VolumeID. Once $VolumeID is defined, we feet the variable into the mountvol utility along with the drive letter we would like the drive to be mounted with.

Likewise, we can use mountvol to unmount the drive once our backup completes.

{% highlight powershell %}
{% raw %}
$EjectDrive = Get-WmiObject -Class Win32_Volume | Where-Object {$_.label -like "<ENTER BACKUP DRIVE NAME OR PARTIAL NAME HERE>"}
$EjectDriveLetter = $EjectDrive.DriveLetter
mountvol $EjectDriveLetter /p
{% endraw %}
{% endhighlight %}

Our first line defines the drive to eject based on the drive's Label property - in my case, all of the local USB drives contained the word "backup" in their label, so I filtered based on that. You could easily reuse the $VolumeID variable and find your drive based on it's DeviceID as well. Our second line defines selects the DriveLetter property of our $Drive and assigns it to the $DriveLetter variable, which we then supply to mountvol alongside the /p flag to eject the drive.

So now our script can: 
  * Mount a drive
  * Perform a backup, 
  * Wait for the backup to complete
  * Unmount the drive.
  
This statisfies requirements 1, 2, and 3. We've also designed the solution using native Windows 10 functionality, so we know we've covered 4 and 5 as well. Additionally, since all we've invested is our time (which we all know has no value at all), we can check requirement 6 off our list as well. Excellent! We've met all of the requirements set out for us. That means we're done, right?

### Logging and Error Handling
Well, sort've. We also need to make sure it works, and that we can fix it when it doesn't. It's frustrating when you come across a solution like this is undocumented and a bit arcane -  it's infuriating when you built it.

With that in mind, we're going to add some logging, error handling, and notification functionality to our script.

We'll start with the New-LogObject function at the top of the script:

{% highlight powershell %}
{% raw %}
Function New-LogObject {
[CmdletBinding()]

  Param(
    [Parameter(Mandatory)]
    [object]$Text,

    [Parameter(Mandatory)]
    [object]$Status
  )

  begin {}
  process {
    $LogObject = [PSCustomObject]@{
      Text = $Text
      Status = $Status
      Timestamp = Get-Date
      }
   }
  end {
    Write-Output $LogObject
}
{% endraw %}
{% endhighlight %}

This function is very simple - it creates a PSCustomObject with a Text, Status, and Timestamp property. We can assign whatever values we want to $Text and $Status. $Timestamp gets defined at time of object creation.

Next, we need a place to put all of the log objects that we create, so let's create a Generic List:

{% highlight powershell %}
{% raw %}
$LogResults = New-Object System.Collections.Generic.List[System.Object]
{% endraw %}
{% endhighlight %}

Generic Lists function more or less like arrays, but have a couple of key differences, on of which we're going to leverage here to improve performance - the .Add() method. .Add() allows us to quickly add new objects to the existing list as we create them without waiting for $LogResults to rebuild like we would if we were using an array with the += operator.

Let's see what this looks like in the context of our script:

{% highlight powershell %}
{% raw %}
$Step = "Mount Volume"
$VolumeID = '<Insert Volume GUID string>'
mountvol E: $VolumeID
$LogResults.Add((New-LogObject -Text $Step -Status "Complete"))
{% endraw %}
{% endhighlight %}

So we define our $Step, perform our action (in this case, mounting the volume), and then once the action is complete, create a New-LogObject with a Status of "Complete" and use the .Add() method to add it to $LogResults. If we add these two lines to each step of our script, we can use $LogResults to determine if individual steps are/not completing.

And what if those steps aren't completing? The code above would still create a log entry with a Status of "Complete", which is why we wrap our script in a try..catch block. I won't dive too deep into the specifics of try..catch, which could be it's own article. Instead, I will provide a link to [the documentation](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_try_catch_finally?view=powershell-6) and explain how it works in the context of our script.

{% highlight powershell %}
{% raw %}
try {
  #Mount the volume and log the results to $LogResults
  $Step = "Mount Volume"
  $VolumeID = '<Insert Volume GUID string>'
  mountvol E: $VolumeID
  $LogResults.Add((New-LogObject -Text $Step -Status "Complete"))
  #REST OF CODE  
  }
catch {
  Write-Verbose "Error encountered. Error code: $Error"
  $LogResults.Add((New-LogObject -Text $Step -Status $Error))
}
$LogResults | Out-File C:\Temp\Results.log
{% endraw %}
{% endhighlight %}

I've omitted most of the script, because I want to focus on the try..catch blocks specifically. Per the documentation linked above, if any point our code inside the try{} block encounters a terminating error, the script will immediately stop it and execute the catch{} block, which contains the automatic variable $Error, which in turn contains the details of our terminating error.

What this means for us is that, if and when our script encounters an error, the catch{} block will grab it and log it into $LogResults alongside current $Step where we encountered the error. $LogResults then immediately gets exported to a file for us to review at our leisure.

But how do we *know* our script is failing without having to manually check Results.log?

This script uses my [Send-SESEmail](https://github.com/mrmonaghan/posh/blob/master/Send-SESEmail.ps1) function, which is really just a glorified Send-MailMessage that makes use of AWS SES SMTP services, to send email notifications based on the results of the try{} block.

{% highlight powershell %}
{% raw %}
try {
  #The whole script
  $ScriptComplete = $true
  }
catch {
  Write-Verbose "Error encountered. Error code: $Error"
  $LogResults.Add((New-LogObject -Text $Step -Status $Error))
}

$LogResults | Out-File C:\Temp\Results.log
$LogString = $LogResults | Out-String
if ($ScriptComplete) {
  Send-SESEmail -To youremail@domain.com -Subject "Backup Complete" -Body $LogString -Attachment C:\Temp\Results.log
  }
else {
  Send-SESEmail -To youremail@domain.com-Subject "Backup Error on $env:Computername" -Body $LogString -Attachment      C:\Temp\Results.log
}
{% endraw %}
{% endhighlight %}

We start by setting the $ScriptComplete variable at the end of our try{} block, ensuring it doesn't exist until the block has compeleted. Once the block has exited, the script checks for the existance of $ScriptComplete. If it's been defined, it sends the "Backup Complete" email with Results.log attached. If $ScriptComplete is not defined, it indicates that an error was encountered, and the script will send the "Backup Error" email with Results.log attached. Thanks to our catch{} block, this contains the error code that caused the try{} block to terminate, allowing us to start troubleshooting.

I would like to finish this post by saying that in no way to I beleive this to be an ideal backup solution, but if you're in a pinch, it's better than absolutely nothing.
