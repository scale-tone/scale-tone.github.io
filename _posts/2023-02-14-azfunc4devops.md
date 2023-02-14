---
title: AzFunc4DevOps released
permalink: /2023/02/14/azfunc4devops
---
![teaser]({{ site.url }}/images/azfunc4devops/azfunc4devops-logo.svg)
# AzFunc4DevOps released.


Azure DevOps (aka ADO, formerly known [both as TFS and VSTS](https://learn.microsoft.com/en-us/azure/devops/server/tfs-is-now-azure-devops-server?view=azure-devops)) is a highly customizable dev collaboration platform. And that is no surprise: thousands of Microsoft's own employees have been using it (or its various predecessors) for over two decades, so surely they have a lot to say about what works well and what needs to be fine-tuned for each particular team or person. Not to mention millions of developers using it outside Microsoft.

Yet as a regular Azure DevOps user, sooner or later you'd want to automate day-to-day routine operations, orchestrate build/release pipelines in some custom sophisticated way, integrate with external systems, import/export data and so on. To address all those specific scenarios, both Azure DevOps Server and Azure DevOps Services expose an extensive RESTful API, with [various platform-specific SDKs](https://learn.microsoft.com/en-us/rest/api/azure/devops/?view=azure-devops-rest-7.1#client-libraries) provided on top of it.

So for you as a developer, it shouldn't be a problem to write a few lines of code that automate your most annoying everyday ADO activity. Or, for example, generate a custom report for your manager. Or analyze a failed build output to create a custom-tailored bug description. And we even have a great code-first Serverless platform to run those few lines of code - [Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-get-started). The only thing missing is native support for Azure DevOps SDK in Azure Functions.

So here is my attempt to weld these two great technologies together: [AzFunc4DevOps](https://github.com/scale-tone/AzFunc4DevOps#azfunc4devops). Technically, it is a set of Azure DevOps triggers and bindings for Azure Functions. Or a set of Azure Functions triggers and bindings for Azure DevOps (whichever way you prefer :)). Anyway, the idea is to make those lines of code be really just a few lines, make them event-driven, easily deployable to Azure (or, in fact, to wherever you prefer to run your Azure Functions) and of course provide them with all the power of Azure Functions as a platform.

Just to not keep you in suspence any longer, here is a quick example that solves what I am personally get most annoyed with on an Azure DevOps sprint board - aligning the **RemainingWork** field with **OriginalEstimate** and **CompletedWork** fields in a Task:

```
[FunctionName(nameof(AlignRemainingWorkWithCompletedWork))]
[return: WorkItem(Project = "%TEAM_PROJECT_NAME%")]
public static WorkItemProxy Run
(
    // When CompletedWork value changes in a Task
    [WorkItemChangedTrigger
    (
        Project = "%TEAM_PROJECT_NAME%",
        WiqlQueryWhereClause = "[System.WorkItemType] = 'Task'",
        FieldName = "Microsoft.VSTS.Scheduling.CompletedWork"
    )]
    WorkItemChange change
)
{
    var task = change.NewVersion;

    task.RemainingWork = task.OriginalEstimate - task.CompletedWork;

    return task;
}
```

And this (a bit more complicated) piece of code automates creating a Test Suite for each created Bug:

```
[FunctionName(nameof(CreateTestSuiteForBug))]
public static async Task Run(

    // When a bug is created
    [WorkItemCreatedTrigger
    (
        Project = "%TEAM_PROJECT_NAME%",
        WiqlQueryWhereClause = "[System.WorkItemType] = 'Bug'"
    )]
    WorkItemProxy bug,
    [WorkItems(
        Project = "%TEAM_PROJECT_NAME%",
        WiqlQueryWhereClause = "[System.WorkItemType] = 'Test Plan'"
    )]
    IEnumerable<WorkItemProxy> existingTestPlanWorkItems,
    [TestPlan(Project = "%TEAM_PROJECT_NAME%")]
    IAsyncCollector<TestPlanProxy> testPlanCollector,
    [TestSuite(Project = "%TEAM_PROJECT_NAME%")]
    IAsyncCollector<TestSuiteProxy> testSuiteCollector
)
{
    int planId;

    // Checking if a Test Plan for this iteration already exists
    var existingTestPlanWorkItem = existingTestPlanWorkItems
        .FirstOrDefault(wi => wi.AreaPath == bug.AreaPath && wi.IterationPath == bug.IterationPath);

    if (existingTestPlanWorkItem != null)
    {
        planId = existingTestPlanWorkItem.Id.Value;
    }
    else
    {
        // Creating a new Test Plan for this iteration
        var newTestPlan = new TestPlanProxy
        {
            Name = $"Test Plan for {bug.IterationPath}",
            AreaPath = bug.AreaPath,
            Iteration = bug.IterationPath
        };

        await testPlanCollector.AddAsync(newTestPlan);

        planId = newTestPlan.Id;
    }

    // Adding a Test Suite for this bug
    var testSuite = new TestSuiteProxy
    {
        PlanId = planId,
        SuiteType = TestSuiteType.RequirementTestSuite,
        RequirementId = bug.Id.Value
    };

    await testSuiteCollector.AddAsync(testSuite);
}
```

To quickly jump to the list of other sample AzFunc4DevOps Functions [just click here](https://github.com/scale-tone/AzFunc4DevOps/tree/main/samples#azfunc4devops-samples).


Now, to start creating your own Functions with AzFunc4DevOps you, of course, need to first create a Functions project:

```
func init --worker-runtime dotnet
```

and install [AzFunc4DevOps.AzureDevOps](https://www.nuget.org/packages/AzFunc4DevOps.AzureDevOps) NuGet package into it:

```
dotnet add package AzFunc4DevOps.AzureDevOps
```

Yes, the current version 1.0.0 only works with [.NET in-process](https://learn.microsoft.com/en-us/azure/azure-functions/functions-dotnet-class-library?tabs=v4%2Ccmd) Function projects. But there're plans to add support for .NET Isolated and Functions written in other supported programming languages, so stay tuned for that.

Then the next step will be to provide three essential configuration settings: **AzureWebJobsStorage**, **AZFUNC4DEVOPS_AZURE_DEVOPS_ORG_URL** and **AZFUNC4DEVOPS_AZURE_DEVOPS_PAT**. 

**AzureWebJobsStorage** setting needs to be provided, because AzFunc4DevOps internally uses [Azure Durable Functions](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview), which require a storage. It's OK to use [Azurite](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azurite) for local development.

**AZFUNC4DEVOPS_AZURE_DEVOPS_ORG_URL** is your Azure DevOps organization's full URL. E.g. `https://dev.azure.com/my-company-name`.

**AZFUNC4DEVOPS_AZURE_DEVOPS_PAT** is your Azure DevOps Personal Access Token. You can [create one in Azure DevOps portal](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate#create-a-pat), or alternatively you can use [KeeShepherd](https://marketplace.visualstudio.com/items?itemName=kee-shepherd.kee-shepherd-vscode) tool for creating and safely handling it.

Important thing is that the PAT needs to be given all [relevant scopes](https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/oauth?view=azure-devops#scopes), depending on what your Functions are going to be doing. E.g. if your Function reads/writes Work Items, then `vso.work_write` is be needed.

As an alternative to **AZFUNC4DEVOPS_AZURE_DEVOPS_ORG_URL** and **AZFUNC4DEVOPS_AZURE_DEVOPS_PAT** settings you can specify **OrgUrl** and **PersonalAccessToken** properties in every trigger/binding attribute. Those properties (just like any other trigger/binding attribute property) also support **%MY-SETTING-NAME%** syntax, which allows to pass these values as config settings as well. See [the example here](https://github.com/scale-tone/AzFunc4DevOps/blob/main/samples/CloneBugsIntoDifferentOrg.cs#L23). Using **OrgUrl** and **PersonalAccessToken** properties lets your Functions operate with multiple different ADO orgs or service instances.
  
When running locally on a devbox, you would typically provide these config setting values via a `local.settings.json` file. Here is an example:

```
{
  "IsEncrypted": false,
  "Values": {
    "FUNCTIONS_WORKER_RUNTIME": "dotnet",

    "AzureWebJobsStorage": "my-azure-storage-connection-string",

    "AZFUNC4DEVOPS_AZURE_DEVOPS_ORG_URL": "https://dev.azure.com/my-company-name",
    "AZFUNC4DEVOPS_AZURE_DEVOPS_PAT": "my-azure-devops-personal-access-token"
  }
}
```
  
When deployed to Azure, those values should be specified via [your Function instance's Application Settings](https://learn.microsoft.com/en-us/azure/azure-functions/functions-how-to-use-azure-function-app-settings?tabs=portal#settings).

Once it is all configured, you can start adding Functions. Which can be triggered by [any of the built-in Triggers](https://learn.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings?tabs=csharp#supported-bindings) or by [any of the Triggers provided by AzFunc4DevOps](https://github.com/scale-tone/AzFunc4DevOps/wiki#triggers). So far AzFunc4DevOps gives you the following Triggers:

* [When a Work Item (Bug, Task, User Story etc.) is created](https://github.com/scale-tone/AzFunc4DevOps/wiki/Reference#T-AzFunc4DevOps-AzureDevOps-WorkItemCreatedTriggerAttribute)
* [When a Work Item (Bug, Task, User Story etc.) changes](https://github.com/scale-tone/AzFunc4DevOps/wiki/Reference#T-AzFunc4DevOps-AzureDevOps-WorkItemChangedTriggerAttribute)
* [When a Build status changes](https://github.com/scale-tone/AzFunc4DevOps/wiki/Reference#T-AzFunc4DevOps-AzureDevOps-BuildStatusChangedTriggerAttribute)
* [When a Pull Request status changes](https://github.com/scale-tone/AzFunc4DevOps/wiki/Reference#T-AzFunc4DevOps-AzureDevOps-PullRequestStatusChangedTriggerAttribute)
* [When a Release Environment (aka Release Stage) status changes](https://github.com/scale-tone/AzFunc4DevOps/wiki/Reference#T-AzFunc4DevOps-AzureDevOps-ReleaseEnvironmentStatusChangedTriggerAttribute)

And of course, your code can then utilize any of the provided Input/Output Bindings, like e.g. a [filtered enumeration of Work Items](https://github.com/scale-tone/AzFunc4DevOps/wiki/Reference#T-AzFunc4DevOps-AzureDevOps-WorkItemsAttribute) or a [collection of Test Cases](https://github.com/scale-tone/AzFunc4DevOps/wiki/Reference#T-AzFunc4DevOps-AzureDevOps-TestSuiteAttribute). [See the extensive list of them documented here](https://github.com/scale-tone/AzFunc4DevOps/wiki#bindings).

Once the code is done, you can then host and run it anywhere where a normal Azure Functions project can be run, ranging from your local devbox [up to a Kubernetes cluster](https://learn.microsoft.com/en-us/azure/azure-functions/functions-kubernetes-keda). But the most natural choice would be to [deploy it into Azure as a Function App instance with Consumption pricing tier](https://learn.microsoft.com/en-us/azure/azure-functions/consumption-plan).

So that's basically what you get from the current [AzFunc4DevOps version 1.0.0](https://www.nuget.org/packages/AzFunc4DevOps.AzureDevOps). But certainly there're lots of plans for the future:

* Add support for .NET Isolated and [all other language workers](https://learn.microsoft.com/en-us/azure/azure-functions/supported-languages), so that the code can be written in any supported language.
* Extend the list of Triggers and Bindings further.
* Implement similar functionality for GitHub, which promises to unveil even more exciting scenarios like near-realtime two-way synchronization, smooth migration etc. 

Please, do share your feedback and ideas on what can be improved and what specific scenarios are yet to be covered [via the project's Discussions board](https://github.com/scale-tone/AzFunc4DevOps/discussions). And of course any sort of contribution or support is extremely welcome.

Happy coding!
