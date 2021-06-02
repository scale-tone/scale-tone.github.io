---
title: Making your Durable Function instances unique with overridableExistingInstanceStates.
permalink: /2020/07/02/making-durable-function-instances-unique
---
![teaser]({{ site.url }}/images/functions/teaser2.png)
# Making your Durable Function instances unique.

**UPD 02.06.2021: It turned out that my original understanding was incorrect in several ways**:
1. The default value for `DurableTaskOptions.OverridableExistingInstanceStates` setting is in fact [OverridableStates.NonRunningStates](https://github.com/Azure/azure-functions-durable-extension/blob/main/src/WebJobs.Extensions.DurableTask/Options/DurableTaskOptions.cs#L163), so you don't actually need to add anything to your `host.json` to get the default "create-if-not-exists" behaviour.
2. Sadly, the above setting still didn't guarantee anything. [There was a race condition in DurableTask extension](https://github.com/Azure/durabletask/pull/528), which (if your multiple `StartNewAsync()` calls get executed quickly enough) could result in your **first activity function to be executed twice**. This was finally fixed in the recent [Durable Functions v2.5.0](https://github.com/Azure/azure-functions-durable-extension/releases/tag/v2.5.0) release, which now [de-duplicates](https://github.com/Azure/durabletask/pull/528) `ExecutionStarted` events.
3. Even with [Durable Functions v2.5.0](https://github.com/Azure/azure-functions-durable-extension/releases/tag/v2.5.0) in place and `DurableTaskOptions.OverridableExistingInstanceStates` explicitly set to **OverridableStates.NonRunningStates** the **StartNewAsync()** method is not fully deterministic: **it might or might not throw that InvalidOperationException** (depending on how much parallel your multiple calls are). So it seems that the only reasonable course of action would be to just catch and swallow (log) it. Instance uniqueness is guaranteed though.

---

When developing so called [Stateful Services](https://www.youtube.com/watch?v=H0i_bXKwujQ), a very important question is how and when to generate new instances of your state. For you as an architect it means, that you'll have to wisely choose the algorithm of generating unique identifiers for your pieces of state: obviously they should be globally unique, while at the same time the algorithm needs to ensure, that no unwanted duplicates are being created. And actually, GUIDs are not always the best solution here, because in many cases state instance identifiers are also used for partitioning, and random partitioning is not always what you want to get.

Because this question is so very fundamental, most frameworks for creating Stateful Services (e.g. based on [Actor Model](https://en.wikipedia.org/wiki/Actor_model) or e.g. implementing the [Saga](https://en.wikipedia.org/wiki/Long-running_transaction) pattern) take it seriously and provide their own built-in algorithms of identifier generation, while also allowing developers to attach their custom implementations. E.g. [Service Fabric Reliable Actors](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-actors-get-started) have their [ActorId](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-actors-platform#actor-id) class, which can be instantiated out of random numbers, more meaningful strings or even **Int64** values (to have manual control over partitioning). **Saga** implementation in [NServiceBus](https://docs.particular.net/nservicebus/) library has a slightly more sophisticated yet more flexible approach to this - you need to explicitly define the *mapping* between incoming messages and their corresponding saga instances by implementing an abstract [ConfigureHowToFindSaga()](https://docs.particular.net/tutorials/nservicebus-sagas/1-getting-started/#exercise-matching-messages-to-sagas) method. But what's most typical for all these frameworks is that they do ensure, that no any instance of your state will ever be duplicated once already exists. Because obviously you don't want your financial transaction to be ever executed twice or two shipments to be delivered out of one single submitted shopping cart.

For some reason though, [Azure Durable Functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview) initially went their own way. The [StartNewAsync()](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.webjobs.extensions.durabletask.idurableorchestrationclient.startnewasync?view=azure-dotnet) method, that you're supposed to use to create new orchestration instances, does can generate unique instance identifiers for you. Or it can take and use an arbitrary string identifier that you generate yourself. But the problem is that if an orchestration instance with such an identifier already exists, the method will "restart" that existing instance. Instead of throwing an error. In a typical scenario of processing a shopping cart and deriving an OrderId from e.g. current browser session this would, of course, mean great chances for a shopping cart to be processed twice (in case a POST request from the client for some reason gets duplicated). Less than ideal...

Fortunately, starting from [Microsoft.Azure.WebJobs.Extensions.DurableTask v2.0.0](https://www.nuget.org/packages/Microsoft.Azure.WebJobs.Extensions.DurableTask/2.0.0) this behavior can now be changed by doing some extra configuration. Specifically, you need to set the [DurableTaskOptions.OverridableExistingInstanceStates](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.webjobs.extensions.durabletask.durabletaskoptions.overridableexistinginstancestates?view=azure-dotnet) config property to **OverridableStates.NonRunningStates** by adding the following line to your **host.json** file:
```
{
    "version": "2.0",
    "extensions": {
        "durableTask": {
            "overridableExistingInstanceStates": 1
        }
    }
}
```

Since then, if your code tries to instantiate an orchestration with the same identifier twice, it will get this **InvalidOperationException**:
    ![image1]({{ site.url }}/images/functions/orchestration-already-exists.png)

NOTE: as of today, this uniqueness is only enforced for *running* orchestrations (those which are in a **Running**, **Pending** or **ContinuedAsNew** state). Which means that if your shopping cart processing is very fast and your orchestration gets completed quickly, then you will still positively surprise some of your customers with duplicated deliveries. To avoid this, you'll need to either explicitly keep your orchestrations running for longer or to use your own [custom distributed lock](https://docs.microsoft.com/en-us/rest/api/storageservices/lease-blob).
