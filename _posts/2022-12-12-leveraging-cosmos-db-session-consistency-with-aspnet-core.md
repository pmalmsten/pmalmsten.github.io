---
layout: post
title:  "Leveraging Cosmos DB Session Consistency with ASP.NET Core"
date:   2022-12-12 22:14:00 -0700
---

Like most fault-tolerant, highly-available NoSQL databases, Cosmos DB offers a
choice of how up-to-date reads are with respect to writes -
including the traditional options of [strong
consistency](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels#strong-consistency)
and [eventual
consistency](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels#eventual-consistency).

However, Cosmos DB also offers additional options between those extremes,
including [consistent
prefix](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels#consistent-prefix-consistency),
[session][session consistency],
and [bounded
staleness](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels#bounded-staleness-consistency),
each providing stronger guarantees than the last.

In this post, we will zoom in on [session
consistency]
\- what makes it a great choice for user-centric web apps / APIs, how it works, and a library
to make it super easy to use with ASP.NET Core.

## What is Cosmos DB Session Consistency?
Imagine Sandy. Sandy uses TweetBookIt™, the hottest new social media app.
Sandy logs in and browses the latest books that other users have up-tweeted.

When Sandy loads the page, her browser doesn't show the latest Michelle Obama
memoir that just achieved Table-of-Contents status less than a second ago. But
that's OK - surely by the time she's caught up on the latest theories for when
*The Winds of Winter* will be finished, the memoir will appear for her. Eventual
consistency is fine for Sandy when she reads her Table-of-Contents.

But when Sandy goes to post her own theory in a new sticky-note, things change -
when she refreshes the page, she expects her sticky-note to still be there, even
just a second after posting it. Eventual consistency is not good enough for her
here - Sandy expects to read-her-own-writes, an element of strong consistency.

Cosmos DB's [session consistency] mode covers both experiences how Sandy
expects, with low cost and high performance similar to eventual consistency -
making it a great fit for TweetBookIt™.

Let's take a look at how session consistency works.

## How Session Consistency Works
As explained by the [Cosmos DB documentation for session consistency][session consistency]:

> In session consistency, within a single client session reads are guaranteed to
> honor the consistent-prefix, monotonic reads, monotonic writes,
> read-your-writes, and write-follows-reads guarantees. This assumes a single
> "writer" session or sharing the session token for multiple writers.

Cosmos DB issues a new session token to a client whenever a write request
finishes - as long as that session token is presented back to Cosmos DB as part
of future read requests, those reads are considered to be part of the same
'session', and they receive the benefits summarized above. Reads issued without
that session token / with a different session token will also eventually reflect
the write, but it may take longer.

The Cosmos DB .NET SDK is smart enough to track session tokens for you
automatically - so when TweetBookIt was still a scrappy startup running from a
single instance of their app, they didn't have to think about session tokens;
everything just worked great:

<pre class="mermaid">
    graph LR;
    Sandy <-- HTTP calls --> TweetBookIt <-- Data + Session Tokens --> CosmosDB;
</pre>

## Scaling Horizontally
Things worked great until TweetBookIt scaled up and put their app behind a load
balancer:

<pre class="mermaid">
    graph LR;
    Sandy <-- HTTP calls --> LoadBalancer <--> TweetBookIt-1 & TweetBookIt-2;
    TweetBookIt-1 <-- Data + Session Tokens for TweetBookIt-1 --> CosmosDB;
    TweetBookIt-2 <-- Data + Session Tokens for TweetBookIt-2 --> CosmosDB;
</pre >

The Cosmos DB .NET SDK still helpfully manages session tokens, but each instance of the SDK in each instance of TweetBookIt™ doesn't know about the session tokens the other instance has managed. So now Sandy's new sticky-note vanishes when she refreshes the page, because her browser's request took a different path through the load balancer this time and ended up using a different Comsos DB session than the one in which she posted her sticky-note a moment ago.

To fix this, we need every request that Sandy issues to TweetBookIt to be in the same Cosmos DB session. A simple way to achieve this is to send the session token to Sandy as a cookie, which she returns with every future request to TweetBookIt™:

<pre class="mermaid">
    graph LR;
    Sandy <-- HTTP calls + Session Tokens --> LoadBalancer <--> TweetBookIt-1 & TweetBookIt-2;
    TweetBookIt-1 <-- Data + Session Tokens for Sandy --> CosmosDB;
    TweetBookIt-2 <-- Data + Session Tokens for Sandy --> CosmosDB;
</pre>

The Cosmos DB SDK can't do this alone - we need to help it track the session
tokens (in this case, using cookies).

## Putting it All Together with ASP.NET Core
In short, for TweetBookIt™ to get the most value out of session consistency, it
needs to:

1. Read the Cosmos DB session token from the cookie (if any) on incoming HTTP
   requests, and attach it to outbound Cosmos DB requests;
2. Keep track of session tokens received from the Cosmos DB .NET SDK while
   processing the client's HTTP request, particularly for writes; and
3. In HTTP responses, update the cookie to have the latest Cosmos DB session
   token value.

We can do these with the Cosmos DB .NET SDK by reading the session token value
returned with Cosmos DB responses from the `response.Headers.Session` field, and
by providing a `RequestOptions` object with the `SessionToken` property set on
outgoing Cosmos DB requests - but that's not easy for web apps or APIs having
10s or 100s of unique paths for reading and writing to/from a database, which
would need extensive plumbing code to pass the session tokens around.

To get the most out of session consistency without the plumbing, I published the
(currently pre-release) `CosmosDB.Extensions.SessionTokens.AspNetCore` library
[(nuget)](https://www.nuget.org/packages/CosmosDB.Extensions.SessionTokens.AspNetCore)
[(github)](https://github.com/pmalmsten/cosmosdb-extensions-sessiontokens-aspnet),
which uses an aspect-oriented approach to avoid the clutter. 

To use the library, simply:

1. Register services in your app's DI container:

    ```c#
    builder.Services.AddCosmosDbSessionTokenTracingServices();
    ```

2. Add session token tracing to any/all `CosmosClient`s you create:  
   
    ```c#
    builder.Services.AddSingleton(provider => 
        new CosmosClient(/*...*/)
            .WithSessionTokenTracing(provider));
    ```

3. Use the ASP.NET Core Middleware for passing session tokens to and from HTTP
   cookies:  
    ```c#
    app.UseMiddleware<CosmosDbSessionTokenCookiesMiddleware>();
    ```

That's it! For every HTTP request to your web app / API, Cosmos DB session
tokens will be automatically propagated to callers through cookies (having names
like `csmsdb-<integer>`). Separate session tokens are tracked for every unique
Comsos DB account/database/container combination used by the app, and calls to
the Cosmos DB SDK are automatically associated with the HTTP request they were
made for using `IHttpContextAccessor`. There are [a few
limitations](https://github.com/pmalmsten/cosmosdb-extensions-sessiontokens-aspnet#limitations)
worth being mindful of, but in many cases it will just work.

Here's to the next TweetBookIt™.

[session consistency]: https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels#session-consistency
