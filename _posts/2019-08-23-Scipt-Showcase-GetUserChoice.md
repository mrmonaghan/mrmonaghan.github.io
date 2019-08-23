---
layout: post
title: Get-UserChoice
---

Occasionally I make tools for other people, which is simultaneously exciting and frustrating. On one hand, I'm able to streamline processes for my colleagues and clients; on the other, I actually have to make the damn things digestable for someone other than myself.

To that end, I've developed the Get-UserChoice function that leverages the '[System.Management.Automation]' class to allow a user to select from a list of options you present them.


{% highlight powershell %}
{% raw %}
function Get-UserChoice {
[CmdletBinding()]

param(
    [Parameter(
        Mandatory = $True,
        ValueFromPipeline = $true,
        ValueFromPipelineByPropertyName = $True)]
    [Alias('Name','Input')]
    [Object]$Data,

    [Parameter()]
    [string]$Title,

    [Parameter()]
    [string]$Description
    )

    Begin {
        $DefaultChoiceIndex = -1
        $Options = @()
        $i = 0
        }
    Process {
        foreach ($value in $Data) {
            $i++
            $NewChoice = [System.Management.Automation.Host.ChoiceDescription]::new("&$value")
            $NewChoice | Add-Member -MemberType NoteProperty -Name Name -Value $value
            $Options += $NewChoice
            }
        }


    end {
        $ChosenIndex = $host.UI.PromptForChoice($Title,$Description,$Options,$DefaultChoiceIndex)
        $Chosen = $Options[$ChosenIndex].name
        Write-Output $Chosen
        }
}
{% endraw %}
{% endhighlight %}

You can leverage this in a number of ways. The simplest is to create a variable that a user can define from a list of values you preselect, like so:

{% highlight powershell %}
{% raw %}
$Letter = Get-UserChoice -Data A,B,C,D -Description "Pick your favorite letter!"
{% endraw %}
{% endhighlight %}

Which would output the following prompt to the user:

![_config.yml]({{ site.baseurl }}/images/blogimages/Get-UserChoice_EX1.png)

Now we have a $Letter variable that contains the user's selection, which we can use further down the line. One way I've been leveraging this functionality is by pairing it with switch{} statements. Building on our earlier example:

{% highlight powershell %}
{% raw %}
$Letter = Get-UserChoice -Data A,B,C,D -Description "Pick your favorite letter!"

switch ($Letter) {
        A {
            Write-Host "You selected option A!" -ForegroundColor Green
        }
        B {
            Write-Host "You selected option B!" -ForegroundColor Red
        }
        C {
            Write-Host "You selected option C!" -ForegroundColor Blue
        }
        D {
            Write-Host "You selected option D!" -ForegroundColor Pink
        }
}
{% endraw %}
{% endhighlight %}

Though simplistic, the above snippet demonstrates how you can build some interactivity into your scripts and very easily allow the end user to make controlled decisions about what your script will do.

"But this is Powershell!" you cry, "What of the objects?"

Fret not, anonymous friend; in addition to the manually defined input in the previous examples, Get-UserChoice also accepts objects and (probably more importantly) objects from the pipeline. This allows us to leverage the command when we're generating a list of options dynamically:

![_config.yml]({{ site.baseurl }}/images/blogimages/Get-UserChoice_EX2.png)
  
In this way, the function can be worked into much more complex scripts. I've created a short trivia game that utilizes Get-UserChoice to allow the user to answer multiple choice questions I've defined in $Data, and a simple if(..) statment to determine whether the user's selection was right or wrong based on my pre-defined Correct property.

{% highlight powershell %}
{% raw %}
$Data = Import-CSV -Path C:\Temp\Data.csv
$Points = 0
$TotalPossible = $Data.QuestionText.Count

foreach ($Question in $Data) {
       $Options = @($Question.A,$Question.B,$Question.C,$Question.D)
       $Selection = New-ChoiceObject -Data $Options -Title $Question.QuestionText -Description "Select one of the below options:"
       Write-Host "You selected $Selection."
       if ($Selection -eq $Question.Correct) {
            $Points++
            Write-Host "Correct!" -F Green
       }
        else {
                Write-Host "You're wrong!" -F Red
        }
}

Write-Host "You earned $Points points out of a possible total of $TotalPossible!"
{% endraw %}
{% endhighlight %}

And the output that is displayed to the user:

![_config.yml]({{ site.baseurl }}/images/blogimages/Get-UserChoice_EX3.png)

As you can see, this is a very robust function. I'm only really beginning to scratch the surface of potential use cases as I dive deeper into Powershell tool-making. Hopefully you can make use of it as well!
