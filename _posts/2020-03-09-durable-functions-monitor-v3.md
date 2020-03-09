---
title: Durable Functions Monitor v3.0 released.
permalink: /2020/03/09/durable-functions-mointor-v3
---
![teaser]({{ site.url }}/images/dfm/vscode-tree-view.png)
# Durable Functions Monitor v3.0 released.

This is just to let you know, that version 3.0 of [Durable Functions Monitor](https://github.com/scale-tone/DurableFunctionsMonitor) is now available. To remind briefly, **Durable Functions Monitor** is a graphical tool for navigating and managing your [Azure Durable Functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=csharp). Since [its start half a year ago](https://scale-tone.github.io/2019/07/26/introducing-durable-functions-monitor) the project already went through a long journey of improvements. Specifically, as of today, it supports both [Durable Orchestrations](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-orchestrations?tabs=csharp) and [Durable Entities](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-entities?tabs=csharp), and is also available in form of a [VsCode extension](https://marketplace.visualstudio.com/items?itemName=DurableFunctionsMonitor.durablefunctionsmonitor).

And the major improvement of v3.0 is that with VsCode extension you can now connect to and manage multiple Task Hubs. The list of currently attached Task Hubs is now displayed in form of a **DURABLE FUNCTIONS** Tree View, which you can find on [Azure Functions](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurefunctions) View Container. **Attach to Task Hub...** button allows you to connect to other Task Hubs, by specifying other Azure Storage connection strings and Hub names. The Tree View is navigatable, so you can keep multiple **Durable Function Monitor** tabs open and switch betweeen them. Context menu also allows attaching/detaching the Task Hubs (this starts/stops the corresponding backend **func.exe** instance) and trigger the **Purge History...** tool for a particular Task Hub. Also, if your currently opened project is a Durable Functions project, the Tree View will be pre-populated with its Task Hub, so you can just click on it to connect.

Plans are to automatically discover and populate the **DURABLE FUNCTIONS** Tree View with all Task Hubs in a certain Azure Storage account, and then eventually to allow deleting unneeded Task Hubs from there.

So, please, enjoy, and I would be very grateful for any support and contribution to [the project](https://github.com/scale-tone/DurableFunctionsMonitor)!