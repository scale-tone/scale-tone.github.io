---
title: Introducing Durable Functions Monitor.
permalink: /2019/07/26/introducing-durable-functions-monitor
---
![teaser]({{ site.url }}/images/dfm/main-page.png)
# Introducing [Durable Functions Monitor](https://github.com/scale-tone/DurableFunctionsMonitor).

UPD: now also available [as a VsCode extension](https://marketplace.visualstudio.com/items?itemName=DurableFunctionsMonitor.durablefunctionsmonitor).

As you know, [Azure Durable Functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview) give you an elegant way of building Reliable Stateful Services for the Cloud, while staying beautifully serverless. In fact, they let you apply advanced architectural patterns, known as [Saga](https://microservices.io/patterns/data/saga.html) (sometimes being called a Workflow), or [Process Manager](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/jj591569(v=pandp.10)) (pretty close to Sagas), or even [Actor Model](https://doc.akka.io/docs/akka/current/guide/actors-intro.html) with minimum efforts, so that your code looks simple and understandable. And it is the underlying framework that splits your code into pieces, that might end up being executed on different VMs, retried and even compensated on failures. 

(Btw. to get a quick understanding of what a "Reliable Stateful Service" typically is, you could take a look at [this demo project](https://github.com/scale-tone/LetsGoOutDemo) of mine.)

That's all great, but without an extensive monitoring/management UI it gets a bit difficult to understand what's going on under the bonnet and particularly to investigate issues. So here is my [Durable Functions Monitor](https://github.com/scale-tone/DurableFunctionsMonitor) - a web UI tool aimed to help you with exactly that.

The tool is itself an Azure Function, but is intended to be run on your local devbox. It has a backend (which is just a thin wrapper around [Durable Functions Management Interface](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-instance-management), and is [written in TypeScript](https://azure.microsoft.com/en-us/blog/improving-the-typescript-support-in-azure-functions/)) and a web UI (React+MobX+TypeScript+Material UI) that lets you swipe through your orchestration instances, sort/filter/auto-refresh them and also monitor/manage each particular instance in a separate browser tab. The way you run it is by just starting the backend Azure Function project and then navigating to **http://localhost:7072/api/monitor** with your favourite browser. The pre-compiled UI static artifacts are served by the same backend. Please, see more detailed instructions on how to do the first run [on github](https://github.com/scale-tone/DurableFunctionsMonitor).

Upon installing the tool on your machine you effectively get all the sources, so feel free to customize them according to your needs. As I said, the pre-compiled UI statics are intentionally committed into codebase (as building them at first run takes forever), but you can always rebuild them with **npm run build**.

Also, technically, nothing prevents you from deploying your Monitor to Azure. E.g. here is me running it under my Azure Function App instance: 
![teaser]({{ site.url }}/images/dfm/in-azure.png)

**Except that so far there is no any authentication implemented in the code**. So it would be entirely your responsibility to protect the deployed app (both the statics and the backend endpoints) from unauthorized access, e.g. by [configuring Easy Auth with AAD](https://docs.microsoft.com/en-us/azure/app-service/overview-authentication-authorization) and [validating user's roles](https://docs.microsoft.com/en-us/azure/architecture/multitenant-identity/app-roles). There're definitely plans to implement this feature, but that's not done yet.

So enjoy it. And everybody is absolutely welcome to provide feedback, report issues, submit feature requests and contribute!

UPD: lots of things have changed since then, especially:
* It now [does support AAD auth](https://github.com/scale-tone/DurableFunctionsMonitor/tree/master/durablefunctionsmonitor.dotnetbackend#how-to-run), so you can safely host it in Azure.
* It is now also available as a [VsCode extension](https://marketplace.visualstudio.com/items?itemName=DurableFunctionsMonitor.durablefunctionsmonitor).
* It now can also show your [Durable Entities](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-entities).
* The backend is now in C#.

Please, check the latest updates [here](https://github.com/scale-tone/DurableFunctionsMonitor/releases).