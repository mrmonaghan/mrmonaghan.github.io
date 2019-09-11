---
layout: post
title: Copy-ADUserPermissions Build Walkthrough
excerpt_separator: <!--more-->
---

Recently I completed a function for a team member who found themselves frequently encountering situations in which they needed to copy AD group membership from one user to another, but were unable to use Active Directory's copy functionality for reasons.

Not only would I like to share this function (as I'm sure my colleague is not the only one suffering the pain of manually adding dozens of group memberships), but I'd also like to break down the process of building and refining it. It is my hope that documenting this process will assist other scripters on their pilgrimage towards writing advanced functions.
<!--more-->

### Step 0: Roadmapping and Building the Framework
First, we need to determine what our function needs to do. I like to map this out visually on a whiteboard, but it's just as easy to put it down in a list format. Minimally, our function will need to:

  1. Get a list of groups our source user is a member of
  2. Get our target user
  3. Apply membership of those groups to our target user
  
That seems fairly straightforward. We can leverage a couple of cmdlets from the Active Directory module and a foreach() loop to achieve something like this:

{% highlight powershell %}
{% raw %}
$SourceGroups = Get-ADPrincipalGroupMembership UserA
$Target = Get-ADUser UserB
foreach ($Group in $SourceGroups) {
  Add-ADGroupMember -Identity $Group -Members $Target
}  
{% endraw %}
{% endhighlight %}

We need to utilize a foreach() loop here because the -Identity parameter of Add-ADGroupMember doesn't accept arrays or collections as a source of input. As a result, we need to iterate over $SourceGroups one $Group at a time, adding our user to each.

Now, let's functionalize it.

### Step 1: Functionalization
There is a lot of literature on the structure of functions and advanced functions (I particularly like [this writeup](https://adamtheautomator.com/powershell-functions/) by Adam Bertram, as it breaks down the structure syntax quite nicely), so I will not dive too deeply into the basics of function structure. Suffice to say, our short script now looks much more intimidating.

{% highlight powershell %}
{% raw %}
Function Copy-ADUserPermissions {
    [CmdletBinding()]

    Param(                        
        [Parameter(Mandatory)]
        [object]$From,

        [Parameter(Mandatory)]
        [object]$To
    )

    begin {
        $SourceGroups = Get-ADPrincipalGroupMembership $From
        $TargetUser = Get-ADUser $To
        }

    }
    process {
        foreach ($Group in $SourceGroups) {
            Add-ADGroupMember -Identity $Group -Members $TargetUser
        }
    }
    end {
        Write-Output "Complete!"
    }
}
{% endraw %}
{% endhighlight %}


Note that I have added two mandatory parameters that a user of this function will need to supply values to, $To and $From. Input to these parameters will define what user we want to use as the source for our copy function, and what user we would like to apply the group memberships to.

You will also see that I've adjusted our variable names a bit in order to make them more descriptive. This will help us out hugely later, when the function starts to become more complex and harder to read.

When a user executes our new function, the workflow now looks something like this:

  1. User runs the function, supplies values to -From and -To 
  2. $SourceGroups and $TargetUser are defined in the begin block using the values supplied by the user
  3. The process block executes our foreach() loop, iterating through $SourceGroups and adding the user to each $Group
  4. The end block displays the output "Complete!" when the process has finished.
  
This covers the barebones requirements we listed above. Are we done?

You know we're not.

### Step 2. Adding Functionality
Now that we have our function doing what we want it to do, let's take a moment to consider some scenarios in which it might be used. There's the situation that started the whole process, needing to copy an existing user's group memberships to a blank user object. But what if we want to copy those memberships to *multiple* blank user objects? What if we want to copy them to a user object that already exists, and has existing memberships?

Let's start by examining what we've built, and how it can accomadate the above scenarios.

#### Scenario 1: Copying group memberships to multiple users
Conviently, our function already supports this natively. Strange though it may seem, while Add-ADGroupMember's -Identity parameter does not accept arrays or collections, its -Members parameter accepts them just fine. Likewise, our -To parameter will accept any number of objects we supply it. We just need to tell it what to do with them.

{% highlight powershell %}
{% raw %}
Function Copy-ADUserPermissions {
    [CmdletBinding()]

    Param(                        
        [Parameter(Mandatory)]
        [object]$From,

        [Parameter(Mandatory)]
        [object]$To
    )

    begin {
        $SourceGroups = Get-ADPrincipalGroupMembership $From
        $TargetUser = Get-ADUser $To
        $TargetUsersArray = New-Object System.Collections.Generic.List[System.Object]
        foreach ($User in $To) {
            $TargetUsersArray.Add((Get-ADUser $User))
        }
    }

    }
    process {
        foreach ($Group in $SourceGroups) {
            Add-ADGroupMember -Identity $Group -Members $TargetUsersArray
        }
    }
    end {
        Write-Output "Complete!"
    }
}
{% endraw %}
{% endhighlight %}

We're going to do this by adding some code to our begin block. We define $TargetUserArray as an empty Generic List, and then use another foreach() loop to iterate through the values supplied to our -To parameter and add the resulting AD user objects to $TargetUserArray. Once that's done, we just need to change the variable we supply to the -Members parameter of Add-ADGroupMember from $TargetUser to $TargetUsersArray.

Hooray! Our function will now copy permissions to as many users as are supplied to the -To parameter!

#### Scenario 2: Overwriting existing group memberships
Now we need to figure out how to clear existing group membership. Thankfully, the Active Directory Powershell module has a cmdlet that should make this pretty easy - Remove-ADGroupMember. We've already done the heavy lifting of building $TargetUsersArray to house all of our target users, so we just need to iterate through that and remove and existing group memberships before we apply the copied memberships. Our code now looks like this:

{% highlight powershell %}
{% raw %}
Function Copy-ADUserPermissions {
    [CmdletBinding()]

    Param(                        
        [Parameter(Mandatory)]
        [object]$From,

        [Parameter(Mandatory)]
        [object]$To,
        
        [Parameter()]
        [switch]$OverwriteExisting
    )

    begin {
        $SourceGroups = Get-ADPrincipalGroupMembership $From
        $TargetUser = Get-ADUser $To
        $TargetUsersArray = New-Object System.Collections.Generic.List[System.Object]
        foreach ($User in $To) {
            $TargetUsersArray.Add((Get-ADUser $User))
        }
    }

    }
    process {
        if ($PSBoundParameters.ContainsKey('OverwriteExisting')) {
            foreach ($User in $TargetUsersArray) {
                Get-AdPrincipalGroupMembership -Identity $User | Where-Object -Property Name -Ne -Value 'Domain Users' | Remove-      AdGroupMember -Members $User -confirm:$false
            }
        }
        foreach ($Group in $SourceGroups) {
            Add-ADGroupMember -Identity $Group -Members $TargetUsersArray
        }
    }
    end {
        Write-Output "Complete!"
    }
}
{% endraw %}
{% endhighlight %}

Most of the changes have occured in the process block with the addition of an if() statement and a foreach() loop. You'll also note that we've added an $OverwriteExisting switch parameter so that we can toggle the overwrite functionality.

If the -OverwriteExisting parameter is present when the function is called, our new foreach() loop will be activated, and begin iterating through $TargetUserArray and removing existing group membership. I've chosen to exclude the Domain Users group from removal, as it both an Active Directory default group and (likely) the primary group of the user, which we cannot remove.

If the -OverwriteExisting parameter is not present when the function is called, the function will simply apply the copied group membership on top of the target user's existing group membership.

### Step 3: Error logging and meaningful output
Something that I find myself asking (and probably writing) a lot is "But how do we know it's working?"

In our function's current iteration, we don't without manually reviewing the membership of each user after running the function. Yikes.
Thankfully, we've already laid some groundwork for implementing error handling, as well as providing more detailed output.


{% highlight powershell %}
{% raw %}
Function Copy-ADUserPermissions {
    [CmdletBinding()]

    Param(
        [Parameter(Mandatory)]
        [object]$From,

        [Parameter(Mandatory)]
        [object]$To,

        [Parameter()]
        [switch]$OverwriteExisting,

        [Parameter()]
        [switch]$ShowErrors

    )

    begin {
        $SourceGroups = Get-ADPrincipalGroupMembership $From
        $TargetUsersArray = New-Object System.Collections.Generic.List[System.Object]
        $ResultsArray = New-Object System.Collections.Generic.List[System.Object]
        foreach ($User in $To) {
            $TargetUsersArray.Add((Get-ADUser $User))
        }

    }
    process {
        if ($PSBoundParameters.ContainsKey('OverwriteExisting')) {
            foreach ($User in $TargetUsersArray) {
                Get-AdPrincipalGroupMembership -Identity $User | Where-Object -Property Name -Ne -Value 'Domain Users' | Remove-AdGroupMember -Members $User -confirm:$false
            }
        }
        foreach ($Group in $SourceGroups) {
            try {
                Add-ADGroupMember -Identity $Group -Members $TargetUsersArray
            }
            catch {
                if ($PSBoundParameters.ContainsKey('ShowErrors')) {
                    foreach ($Line in $Error) {
                    Write-Error $Line
                    }
                }

            }
        }

    }
    end {
        #SEE BELOW
    }
}
{% endraw %}
{% endhighlight %}

I've omitted changed to the end block in this example, because I want to discuss the changes to the begin and process block first. In the begin block, we've created a new $ResultsArray Generic List, which we'll use later on to filter and display our detailed output. We've also wrapped the Add-ADGroupMember portion of our process block in a try..catch block. I've briefly discussed try..catch in a [previous post](https://mrmonaghan.github.io/Building-a-Backup-Script-With-WBAdmin/), and the usage here is very similar: as the foreach() loop tries to add members to each $Group, the catch{} block will capture any errors that occur and log them into the automatic $Errors variable. As you can see, I've added a -ShowErrors switch parameter that, if present when the function is run, will write each $Line in $Errors to the console as it occurs.

Next, let's discuss the end block.

{% highlight powershell %}
{% raw %}
end {
        foreach ($User in $TargetUsersArray) {
            $UserResultObject = [PSCustomObject]@{
                UPN = $User.SamAccountName
                Status = $null
                DiffGroup = $null
                SourceOrTarget = $null
            }
            $TargetUserGroupMembership = Get-ADPrincipalGroupMembership $User
            $Comparison = Compare-Object -ReferenceObject $SourceGroups -DifferenceObject $TargetUserGroupMembership
            if (!($Comparison)) {
                $UserResultObject.Status = 'Success'
                }
            else {
                $UserResultObject.Status = 'Error'
                $UserResultObject.DiffGroup = $Comparison.InputObject.name
                $UserResultObject.SourceOrTarget = $Comparison.SideIndicator
                }
            $ResultsArray.Add($UserResultObject)
        }
        Write-Output $ResultsArray
    }
{% endraw %}
{% endhighlight %}

This is where most of the changes have occured since our last iteration. Our new end block will:

  1. Loop through $Users in $TargetUsersArray and create a PSCustomObject called $UserResultObject for each one
  2. Assign values to the keys in $UserResultObject
  3. Get a list of each $User's current group membership, now that the process block has finished adding each $User to $SourceGroups
  4. Use Compare-Object to check $User's group membership against $SourceGroup. If they are the same, sets the Status value of    $UserResultsObject to 'Success'. If they are not the same, sets the Status value of $UserResultsObject to 'Error' and adds values to the two other properties of $UserResultObject - one that contains all of the groups that were not shared between $TargetUserGroupMembership and $SourceGroup, and one that indicates whether the membership comes from the source user or the target user.
  6. At the end of the foreach() loop, the complete $UserResultObject is added to the $ResultsArray that we created earlier.
  7. Once the foreach() loop has completed, uses Write-Output to display $ResultsArray to the user.
  
That's a lot to process. Thankfully, it's much clearer when we see it in action.
  
Here we can see the results of two successful membership copies:

![_config.yml]({{ site.baseurl }}/images/blogimages/Copy-ADUserPermissions_EX1.png)
  
And here we can see the results of the function when a target user's group membership is not the same as the $SourceGroup array:

![_config.yml]({{ site.baseurl }}/images/blogimages/Copy-ADUserPermissions_EX2.png)

We can see that the DiffGroup in question is Group5, and that RWeasley is a member of it. This makes sense, as we did not run the function with the -OverwriteExisting paramter, so the $SourceGroup membership would have been applied on top of whatever groups RWeasley was already in.
  
### Conclusion
The full function is available for review and download [at its dedicated repository, here.](https://github.com/mrmonaghan/Copy-ADUserPermissions) It's my hope that, through this description of process or the function itself, I'm able to provide a solution to a problem someone is furiously googling.
