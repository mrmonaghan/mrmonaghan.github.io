---
layout: post
title: Building a REST API with Powershell
---

Recently I was tasked with building a simple REST API to serve some data from an on-premise SQL database to an Azure Logic app. At the time, the only word in that sentence I was familiar with was "simple", and since I'm a bit of a one-trick pony when it comes to scripting languages, I decided to build the solution with Powershell. After some internet sleuthing and creative trial and error, I had a working solution that would happily serve the requested data to our Logic App. 

I wanted to outline the process of building the API becuase A) I haven't posted anything in quite some time and B) because there is not a wealth of information out there about how to accomplish something like this with Powershell, and I like to contribute.

Without further ado:

### What is a REST API?

For those of us (read: me a week ago) who are unfamiliar with REST APIs, the ELI5 version is something like "an applicaton that recieves requests from the internet and returns requested information." This is a pretty huge oversimplification, but it's enough to tell us what we need to accomplish - listen for a requests, use the content of the request to find some kind of data, then return that data to the origin of the request.

Neat.

So, how do we listen for a request? A cursory Google search reveals a .NET class that does just that, [HTTPListener](https://docs.microsoft.com/en-us/dotnet/api/system.net.httplistener?view=netframework-4.8). A quick review of the documentation allows us to contruct the following simple HTTPListener:

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

{% endraw %}
{% endhighlight %}
