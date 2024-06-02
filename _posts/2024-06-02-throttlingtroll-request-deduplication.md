---
title: ThrottlingTroll, and how to deduplicate requests with it
permalink: /2024/06/02/throttlingtroll-request-deduplication
---
<img src="{{ site.url }}/images/throttlingtroll/no-double-click.svg" width="400px" style="padding-left:100px">
# ThrottlingTroll, and how to deduplicate requests with it.

Certain HTTP requests should never be processed twice. A well-known example would be an online shopping cart processing request which, if not deduplicated properly, could result in charging the customer a double amount of money.

Knowing about this typical challenge, most developers would apply advanced messaging patterns and employ Message Queuing Services that support duplicate detection (like [Azure Service Bus](https://learn.microsoft.com/en-us/azure/service-bus-messaging/duplicate-detection) or [Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/using-messagededuplicationid-property.html)). Just for the sake of solving this simple problem.

But [ThrottlingTroll](https://github.com/ThrottlingTroll/ThrottlingTroll) now provides an easier solution to ensure Exactly-Once Processing. Its [Semaphore](https://github.com/ThrottlingTroll/ThrottlingTroll/wiki/410.-Rate-Limiting-Algorithms#-semaphore) rate limiter, combined with a *distributed* Counter Store like [ThrottlingTroll.CounterStores.Redis](https://github.com/ThrottlingTroll/ThrottlingTroll/tree/main/ThrottlingTroll.CounterStores.Redis#throttlingtrollcounterstoresredis), allows to organize *distributed named critical sections* (aka named mutexes). And if we set that critical section's name to a request ID of some sort and then *postpone releasing* that critical section for a certain period of time (known as Duplicate Detection Window), the effect will be exactly what we want: the first request with that ID will be processed, while all other requests with the same ID will be rejected. And that request ID can come in any form: it can be a query string parameter, a custom header, a part of request body - the logic is all yours here.

Here is how to configure it. First we need to initialize ThrottlingTroll at our project's startup, and configure a distributed Counter Store and an **IdentityIdExtractor** routine (which should produce a request ID out of each incoming request):

```
app.UseThrottlingTroll(options =>
{
    // Using ThrottlingTroll.CounterStores.Redis as a distributed Counter Store
    options.CounterStore = new RedisCounterStore(ConnectionMultiplexer.Connect(myRedisConnString));

    // Using "id" query string param to identify requests. Adjust the logic as needed.
    options.IdentityIdExtractor = request =>
    {
        return ((IIncomingHttpRequestProxy)request).Request.Query["id"];
    };

    // Returning 409 Conflict for duplicate requests. You may as well choose to return any other status code.
    options.ResponseFabric = async (checkResults, requestProxy, responseProxy, requestAborted) =>
    {
        responseProxy.StatusCode = StatusCodes.Status409Conflict;

        await responseProxy.WriteAsync("Duplicate request detected");
    };
});
```

And then apply the [Semaphore](https://github.com/ThrottlingTroll/ThrottlingTroll/wiki/410.-Rate-Limiting-Algorithms#-semaphore) limiting rule (with 1 parallel request allowed) to our HTTP endpoint:

```
[ApiController]
public class ShoppingCartController : ControllerBase
{
    [HttpPost("shopping-cart")]

    // Configure ThrottlingTroll to allow only one parallel request and to release the lock after 10 seconds.
    // Note that ReleaseAfterSeconds should be less than TimeoutInSeconds (which is 100 by default)
    [ThrottlingTroll(Algorithm = RateLimitAlgorithm.Semaphore, PermitLimit = 1, ReleaseAfterSeconds = 10, TimeoutInSeconds = 100)]

    public void Post()
    {
        // Process the shopping cart here
    }
}
```

And that's it. Now ThrottlingTroll will automatically reject duplicate requests for us:

<img src="https://scale-tone.github.io/images/throttlingtroll/shopping-cart-409.png" width="400px"/>


To underscore again, for this to work in a distributed cloud-based solution, you would need to use a *distributed* Counter Store. Note that by default ThrottlingTroll uses an in-memory Counter Store, and a different option needs to be configured explicitly, as shown above. [ThrottlingTroll.CounterStores.Redis](https://github.com/ThrottlingTroll/ThrottlingTroll/tree/main/ThrottlingTroll.CounterStores.Redis#throttlingtrollcounterstoresredis) looks like the best choice here, but you might consider [other distributed alternatives](https://github.com/ThrottlingTroll/ThrottlingTroll/wiki/510.-Counter-Stores), or even [implement your own](https://github.com/ThrottlingTroll/ThrottlingTroll/blob/main/ThrottlingTroll.Core/CounterStores/ICounterStore.cs).

Last thing to mention is that ThrottlingTroll works both with ASP.NET Core and Azure Functions (.NET Isolated), so the above approach would work equally well in your Functions project.

[Explore all other numerous ThrottlingTroll's features here](https://github.com/ThrottlingTroll/ThrottlingTroll/tree/main?tab=readme-ov-file#throttlingtroll) and enjoy.