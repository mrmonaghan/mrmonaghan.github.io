---
layout: post
title: Building a REST API with Powershell
---

Recently I was tasked with building a simple REST API to serve some data from an on-premise SQL database to an Azure Logic app. At the time, the only word in that sentence I was familiar with was "simple", and since I'm a bit of a one-trick pony when it comes to scripting languages, I decided to build the solution with Powershell. After some internet sleuthing and creative trial and error, I had a working solution that would happily serve the requested data to our Logic App. 

I wanted to outline the process of building the API becuase A) I haven't posted anything in quite some time and B) because there is not a wealth of information out there about how to accomplish something like this with Powershell, and I like to contribute.

Without further ado:

### What is a REST API?

For those of us (read: me a week ago) who are unfamiliar with REST APIs, the ELI5 version is something like "an applicaton that recieves requests from the internet and returns requested information." This is a pretty huge oversimplification, but it's enough to tell us what we need to accomplish - listen for a requests, use the content of the request to find some kind of data, then return that data to the origin of the request.

Neat. So, how do we listen for a request? 

### Getting The Request

A cursory Google search reveals a .NET class that does just that, [HTTPListener](https://docs.microsoft.com/en-us/dotnet/api/system.net.httplistener?view=netframework-4.8). A quick review of the documentation allows us to contruct the following simple HTTPListener:

{% highlight powershell %}
{% raw %}
# Create The Listener
$listener = New-Object System.Net.HttpListener
$listener.Prefixes.Add('http://+:8000/')
$listener.Start()
{% endraw %}
{% endhighlight %}

And thusly we have created our first listener, which we can send data to using Powershell's Invoke-RESTMethod. In order to actually view and utilize the data our listener recieves, though, we'll need to utilize some of the HTTPListener's methods.

{% highlight powershell %}
{% raw %}
    $Context = $listener.GetContext()
    $Request = $Context.Request
    $body = $request.InputStream
    $Response = $Context.Response
    $encoding = $request.ContentEncoding
    $reader = New-object System.IO.Streamreader($body,$encoding)
    $String = $Reader.ReadToEnd()
{% endraw %}
{% endhighlight %}

The above code sets up the necessary structure to actually retrieve the content of the request out listener recieves, in addition to the objects we will need to leverage to send our response later on. Note that the content of the initial request come encoded, and we must use the $Reader object we create to decode it into plaintext.

Now the body of our REST request will be stored in $String. From here, we can leverage it into whatever kind of function we need. Now we just need to...

### Respond
Finally, we need to provide a response to our request. I have created the following function to make this easier in situations where you will need to response to multiple potential requests:

{% highlight powershell %}
{% raw %}
Function New-RESTResponse {
    [CmdletBinding()]

    Param(
        [Parameter(Mandatory)]
        [object]$Message
    )

    begin {}
    process {
        $MessageJSON = $Message | ConvertTo-JSON
        [byte[]]$buffer = [System.Text.Encoding]::UTF8.GetBytes($MessageJson)
        $response.ContentLength64 = $buffer.length
        $output = $response.OutputStream
        $output.Write($buffer, 0, $buffer.length)
        $output.Close()
    }
    end {}
}
{% endraw %}
{% endhighlight %}

The above function will take the object provided to it in the Message parameter, attempt to convert it to JSON, then rencode it and add it to the $Response object we created earlier. Finally, it writes the output back to the origin of our request.

### In Action
Now that we have the bones of our API built, we need to test. In order to keep the API running indefinitely, we'll wrap the entire script (minus creating the HTTPlistener) in a while.. statment like so:

{% highlight powershell %}
{% raw %}
# create listener
$listener = New-Object System.Net.HttpListener
$listener.Prefixes.Add('http://+:8000/')
$listener.Start()

while($true) {
  All Remaining Code
}
{% endraw %}
{% endhighlight %}

Now we can simply start our listener and begin sending requests using Invoke-RestMethod.
