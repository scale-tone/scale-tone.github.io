---
title: ThrottlingTroll, and how to make distributed locks with it
permalink: /2023/05/22/throttlingtroll
---
<img src="{{ site.url }}/images/throttlingtroll/teaser.svg" width="650px">
# ThrottlingTroll, and how to make distributed locks with it.

Yes, I know that there have been many rate limiting/throttling middleware libraries already created for ASP.NET. 

Most well-known is probably Stefan Prodan's great [AspNetCoreRateLimit](https://github.com/stefanprodan/AspNetCoreRateLimit) project, which provides many advanced features like [storing counters in a distributed cache](https://github.com/stefanprodan/AspNetCoreRateLimit/wiki/Using-Redis-as-a-distributed-counter-store), [limiting clients individually based on their IDs](https://github.com/stefanprodan/AspNetCoreRateLimit/wiki/ClientRateLimitMiddleware#setup), [updating limits at runtime](https://github.com/stefanprodan/AspNetCoreRateLimit/wiki/IpRateLimitMiddleware#update-rate-limits-at-runtime) and more. Yet it only supports one rate limiting algorithm ("fixed window", from what I can see), can only return error status codes (and cannot instead delay/damper responses, for example) and (as the name implies) is only intended for ASP.NET.

In .NET 7 we now have [an out-of-the-box rate limiting middleware](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit?view=aspnetcore-7.0), which offers several different rate limiting algorithms (including [concurrency limiter](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit?view=aspnetcore-7.0#concurrency-limiter)), various ways to limit clients individually (based [e.g. on their IP-addresses](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit?view=aspnetcore-7.0#limiter-with-onrejected-retryafter-and-globallimiter)) and [declarative (attribute-based) configuration](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit?view=aspnetcore-7.0#enableratelimiting-and-disableratelimiting-attributes). Yet there's no way to dynamically reconfigure rate limits without restarting/redeploying the service (which might be a vital feature in the event of a DDoS attack) and there's no built-in support for distributed counter stores (although people have already done supplements to this deficiency, e.g. take a look at Cristi Pufu's [aspnetcore-redis-rate-limiting](https://github.com/cristipufu/aspnetcore-redis-rate-limiting)). And again, it only works in ASP.NET.

So I created [ThrottlingTroll](https://github.com/scale-tone/ThrottlingTroll#throttlingtroll), which is my take on rate limiting/throttling both for ASP.NET _and_ Azure Functions (.NET Isolated). On top of everything it supports: 
* Three different ways of configuring - via a config-file, programmatically at startup and dynamically (aka via a periodically re-executed routine, which is expected to fetch limit values from wherever).
* Both ingress and _egress_ throttling. The latter means that you can configure an HttpClient instance that will limit itself and shortcut to a `429 TooManyRequests` response (without making the actual call) when the limit is exceeded. Also you can configure an HttpClient so, that it automatically propagates `429 TooManyRequests` responses from egress to ingress.
* Distributed counter stores, including **RedisCounterStore**.
* Multiple rate limiting algorithms, including **Semaphore** aka Concurrency limiter.

For a comprehensive list of features, documentation and samples I suggest you check out [ThrottlingTroll's GitHub repo](https://github.com/scale-tone/ThrottlingTroll). Here I just wanted to talk about one interesting application of it - organizing named distributed locks (aka critical sections).

Quite often you might need to protect some server-side state from being corrupt by concurrent requests. A very typical example would be implementing a shopping cart, that should not at any circumstances be processed twice or updated in parallel (while technically both things always have a chance to happen, due to e.g. a network glitch or simply because user is pressing buttons in several browser tabs). This normally requires configuring and implementing a distributed lock. And [ThrottlingTroll](https://github.com/scale-tone/ThrottlingTroll) does this for you.

All that it takes is:
* Install [ThrottlingTroll](https://www.nuget.org/packages/ThrottlingTroll) (for ASP.NET) or [ThrottlingTroll.AzureFunctions](https://www.nuget.org/packages/ThrottlingTroll.AzureFunctions) (for Azure Functions) from NuGet. 
* Configure it as follows:
```
app.UseThrottlingTroll(options =>
{
    options.CounterStore = new RedisCounterStore(
        ConnectionMultiplexer.Connect(builder.Configuration["RedisConnectionString"])
    );

    options.Config = new ThrottlingTrollConfig
    {
        Rules = new[]
        {
            new ThrottlingTrollRule
            {
                UriPattern = "/shopping-cart",

                LimitMethod = new SemaphoreRateLimitMethod
                {
                    PermitLimit = 1
                },

                MaxDelayInSeconds = 120,

                IdentityIdExtractor = request =>
                {
                    return ((IIncomingHttpRequestProxy)request).Request.Query["id"];
                }
            },
        }
    };
});
```

The above code first instructs ThrottlingTroll to use **RedisCounterStore** for storing counters in a shared place (don't forget to put `RedisConnectionString` into your config file).
Then it configures a rate limiting rule, that's being applied to the `/shopping-cart` endpoint. 
That rule uses **SemaphoreRateLimitMethod** with **PermitLimit** set to **1**, which means no more than 1 concurrent request to that endpoint is allowed.
Non-zero **MaxDelayInSeconds** value makes ThrottlingTroll apply spin-wait logic when that limit is exceeded, thus placing all pending requests in a queue (when **MaxDelayInSeconds** is zero, `429 TooManyRequests` are immediately returned instead).
Then finally a custom **IdentityIdExtractor** is used to identify and separate shopping cart instances from each other. Here we assume that they're identified by an `id` query string parameter, but of course it can be anything else (header value, claim value etc.) so long as that value is globally unique.

That's it. Now access to your shopping cart API methods will be synchronized, with no danger of ending up in a corrupt state. And you name all other potential applications for this feature.

In ThrottlingTroll's samples folder you can find a distributed counter implementation - [this one for ASP.NET](https://github.com/scale-tone/ThrottlingTroll/blob/edf507430262a785a55275432d8fb0e113a4062b/samples/ThrottlingTrollSampleFunction/Program.cs#L243) and [this one for Azure Functions](https://github.com/scale-tone/ThrottlingTroll/blob/edf507430262a785a55275432d8fb0e113a4062b/samples/ThrottlingTrollSampleWeb/Program.cs#L266) - demonstrating the very same idea. 
And of course check out all other samples there. 

Enjoy and [suggest further features](https://github.com/scale-tone/ThrottlingTroll/issues), if anything is missing.
