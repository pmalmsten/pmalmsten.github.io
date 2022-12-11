---
layout: post
title:  "Leveraging Cosmos DB Session Consistency with ASP.NET"
date:   2022-12-11 10:00:00 -0700
categories: ASPNET C# CosmosDB
---

Like many other NoSQL databases, Cosmos DB offers a choice between different
consistency levels for reads - including strong consistency (reads always return
the latest version of records, but with lower performance and higher cost), and
eventual consistency (reads may return abitrarily old versions of records, but
higher performance and lower cost). 

Unlike many other NoSQL databases, Cosmos DB offers choices between these
extremes - one such option is session consistency. This mode strikes a balance
between strong consistency and eventual consistency by introducing the concept
of a 'session': reads issued in the same session as writes are guaranteed to
immediately reflect those writes; whereas reads issued in a different session
than writes will eventually reflect those writes, and the writes will be seen in
order, but there is no promise as to when reads will reflect those writes.

This makes session consistency a great choice for applications that have
separate users - each user can immediately read back the writes he or she made,
but without as significant of performance or cost penalties as the strong
consistency mode.

So how can we take advantage of session consistency for an ASP.NET Web App or
Web API? To answer that, we need to peek behind the curtain for how Cosmos DB
sessions and the Cosmos DB .NET SDK work.

## Cosmos DB Sessions and the Cosmos DB .NET SDK
A 'session' in Cosmos DB is defined by a 'session token' that the Cosmos DB
server includes in responses to read and write requests for items. Any
subsequent item read/write request that includes the session token
from a prior response is considered in the same 'session' as the prior request.

Imagine a series of sequential reads and writes issued by a single thread - as
long as the session token value returned from the prior response is passed into
the next request, all of the reads and writes in that sequence are considered as
being in the same 'session'.

To make things easier, the Cosmos DB SDK keeps track of these session tokens for
us automatically:

> When working with Session consistency, each new write request to Azure Cosmos
> DB is assigned a new SessionToken. The CosmosClient will use this token
> internally with each read/query request to ensure that the set consistency
> level is maintained.

So for any single instance of an application (more specifically, the
`CosmosClient` object), we don't have to think about session tokens. But what if
I have more than one instance of my application - say I have a load balancer, and
want to distribute incoming traffic between 3, 5, 10, or more instances of my app?

## Giving Users their own Sessions
If we have exactly one instance of our application, all of our users effectively
share a single Cosmos DB 'session' (thanks to the SDK tracking session tokens
for us), so this works out OK:

<pre class="mermaid">
    graph LR;
    Client-1 <-- API calls --> YourApp <-- Data + Session Tokens --> CosmosDB;
</pre>

However, if we add additional instances of our application behind a load
balancer, this breaks down:

> If you are using a round-robin load balancer which does not maintain session
> affinity between requests, such as the Azure Load Balancer, the read could
> potentially land on a different node to the write request, where the session
> was created.

Since different instances of the SDK don't know about session tokens received by
other instances of the SDK, each instance of the app effectively operates as a 
separate 'session':

<pre class="mermaid">
    graph LR;
    Client-1 <-- API calls --> LoadBalancer <--> YourApp-1 & YourApp-2;
    YourApp-1 <-- Data + Session Tokens for YourApp-1 --> CosmosDB;
    YourApp-2 <-- Data + Session Tokens for YourApp-2 --> CosmosDB;
</pre >

When subsequent requests from a user are routed to a different node, the user
might not see the writes they recently issued, since different nodes are effectively
in different sessions.

To solve this problem, we need to define sessions per user of our app, not per instance
of our app - and to do that, we need to start managing session tokens ourselves:

> Consider a web application with multiple nodes, each node will have its own instance of CosmosClient. If you wanted these nodes to participate in the same session (to be able read your own writes consistently across web tiers) you would have to send the SessionToken from FeedResponse<T> of the write action to the end-user using a cookie or some other mechanism, and have that token flow back to the web tier and ultimately the CosmosClient for subsequent reads.

<pre class="mermaid">
    graph LR;
    Client-1 <-- API calls + Session Tokens --> LoadBalancer <--> YourApp-1 & YourApp-2;
    YourApp-1 <-- Data + Session Tokens for Client-1 --> CosmosDB;
    YourApp-2 <-- Data + Session Tokens for Client-1 --> CosmosDB;
</pre>


You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

Jekyll requires blog post files to be named according to the following format:

`YEAR-MONTH-DAY-title.MARKUP`

Where `YEAR` is a four-digit number, `MONTH` and `DAY` are both two-digit numbers, and `MARKUP` is the file extension representing the format used in the file. After that, include the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
