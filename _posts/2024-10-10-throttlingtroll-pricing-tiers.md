---
title: ThrottlingTroll, and how to make pricing tiers for your API with it
permalink: /2024/10/10/throttlingtroll-pricing-tiers
---
<img src="{{ site.url }}/images/throttlingtroll/pricing-tiers.png" width="650px" >
# ThrottlingTroll, and how to make pricing tiers for your API with it.

Another typical task when building web APIs is to implement *pricing tiers*, aka to apply different request rate limits based on which tariff the client's api-key belongs to.

[ThrottlingTroll](https://github.com/ThrottlingTroll/ThrottlingTroll) can handle that as well. Specifically for that it offers [CostExtractors](https://github.com/ThrottlingTroll/ThrottlingTroll/wiki/150.-%5BIngress%5D-Personalized-rate-limiting#using-costextractors).

The idea is simple. You set the overall rate limit to some big number, e.g. **10000** requests per minute. And then you write a CostExtractor routine, that takes the incoming api-key from the request, maps that api-key to some pricing tier and returns a different request "cost" based on that pricing tier. E.g. for a "silver" tariff the cost could be **100**, for "gold" it could be **10** and **1** for "platinum". As a result, a "silver" client would be limited to **100** requests per minute, a "gold" one would experience **1000** requests per minute and a "platinum" customer would enjoy **10000** request per minute.

(Just to mention, the default request "cost", when no CostExtractors are provided, is of course **1**)

The code would look as simple as this:

```
app.UseThrottlingTroll(options =>
{
    options.Config = new ThrottlingTrollConfig
    {
        Rules = new[]
        {
            new ThrottlingTrollRule
            {
                LimitMethod = new FixedWindowRateLimitMethod
                {
                    PermitLimit = 10000,
                    IntervalInSeconds = 60
                },

                CostExtractor = request =>
                {
                    string? apiKey = ((IIncomingHttpRequestProxy)request).Request.Query["api-key"];

                    var pricingTier = GetPricingTierForApiKey(apiKey);

                    switch (pricingTier)
                    {
                        case PricingTier.Platinum:
                            return 1;
                        case PricingTier.Gold:
                            return 10;
                        case PricingTier.Silver:
                            return 100;
                        default:
                            return 1000;
                    }
                }
            },
        },
    };
});
```

Or you could even [configure the limits reactively](https://github.com/ThrottlingTroll/ThrottlingTroll/wiki/130.-%5BIngress%5D-How-to-configure-reactively) (load them from some storage) and provide your  CostExtractor routine globally, [like shown here](https://github.com/ThrottlingTroll/ThrottlingTroll/wiki/150.-%5BIngress%5D-Personalized-rate-limiting#using-costextractors).

Just remember that both [IdentityIdExtractors](https://github.com/ThrottlingTroll/ThrottlingTroll/wiki/150.-%5BIngress%5D-Personalized-rate-limiting#extracting-and-using-identityids) and [CostExtractors](https://github.com/ThrottlingTroll/ThrottlingTroll/wiki/150.-%5BIngress%5D-Personalized-rate-limiting#using-costextractors) will be called as part of each request processing, so they're supposed to be fast. That's why they're intentionally synchronous. So it would be a good idea to preload and cache that api-key-to-pricing-tier mapping and whatever else data your code might need.

As usual, you can use [ThrottlingTroll.CounterStores.Redis](https://github.com/ThrottlingTroll/ThrottlingTroll/tree/main/ThrottlingTroll.CounterStores.Redis#throttlingtrollcounterstoresredis) to make the limits be applied per your entire service and not per each compute node.

As usual, it all works [both with ASP.NET Core and Azure Functions (.NET Isolated)](https://github.com/ThrottlingTroll/ThrottlingTroll/wiki#installing-from-nuget).