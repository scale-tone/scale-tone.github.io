---
title: How to deploy your Azure Functions from a NuGet package.
permalink: /2020/11/26/deploy-azure-functions-from-nuget-package
---
![teaser]({{ site.url }}/images/functions/teaser4.png)
# How to deploy your Azure Functions from a NuGet package.


Yes, it is absolutely possible:
<script src="https://gist.github.com/scale-tone/234c1f148b4c25678fe0dcbfe2f26126.js"></script>

The above [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/) script creates a new Function App instance and deploys some .Net code into it from a local `my-local-nuget-package.nupkg` file.

As we all know, a NuGet package is in fact [nothing but a ZIP-file](https://en.wikipedia.org/wiki/NuGet) containing whatever you put in it, plus some NuGet-related metadata. Change the package file extension from **.NUPKG** to **.ZIP** - and you'll be able to look inside of it with Windows File Explorer.

On the other hand, Azure Functions do support [so called Zip Deployment](https://docs.microsoft.com/en-us/azure/azure-functions/run-functions-from-deployment-package), aka copying compiled binaries and config files from a provided ZIP-file into Function Instance's `wwwroot` folder. The same is also very true for 'classic' Azure App Services - the **WEBSITE_RUN_FROM_PACKAGE** app setting, [that enables it](https://markheath.net/post/deploy-azure-webapp-kudu-zip-api), was introduced years ago.

Combine the above two knowledges together - and voila, [it just works](https://github.com/scale-tone/DurableFunctionsMonitor/tree/master/durablefunctionsmonitor.dotnetbackend#how-to-run).

Not every NuGet package can be deployed in that way, of course, but only a specially formatted one. E.g. for .Net-based Azure Functions it should contain the output of `dotnet publish` command, and the way I managed to achieve that (there can be other ways as well) is by having a NUSPEC-file like this:

<script src="https://gist.github.com/scale-tone/9e7910960d6803ca0dd6f8ee2e76b3b1.js"></script>

Call that file e.g. `nuspec.nuspec`, put it next to your CSPROJ-file and *include it into your project* (so that it gets copied as well):
```
<None Update="nuspec.nuspec">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
</None>
```
Then in your project root folder run this:
```
dotnet publish
nuget pack bin\Debug\netcoreapp2.1\publish\nuspec.nuspec
```
and a newly created `my-nuget-packed-function.1.0.0.nupkg` file should appear.

For a Function written in e.g. JavaScript/TypeScript the NUSPEC-file will look a bit different:

<script src="https://gist.github.com/scale-tone/7b8b7cf02705d14682ac5c013dd5abc0.js"></script>

(the main difference is that, since this file now resides in the project's root folder, we'll need to exclude some stuff that should never be packed).

And then you run:
```
npm install
npm build
nuget pack nuspec.nuspec
```
to get your JavaScript/TypeScript Function App packed.

NOTE: if you don't have `NuGet.exe` installed, you can get it [from here](https://www.nuget.org/downloads).

Now, ways to trigger the [Zip Deployment](https://docs.microsoft.com/en-us/azure/azure-functions/deployment-zip-push) of your package are actually numerous. Including even [via an HTTP POST](https://docs.microsoft.com/en-us/azure/azure-functions/deployment-zip-push#with-curl). But for me the most reasonable thing to do was to combine it with [ARM templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-syntax), so that my users can avail from that quick and nice [Deploy to Azure](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/deploy-to-azure-button) button to instantly deploy *my* Function App into *their* Azure subscriptions. [The entire working ARM template](https://github.com/scale-tone/DurableFunctionsMonitor/blob/master/durablefunctionsmonitor.dotnetbackend/arm-template.json) would be a bit too long and irrelevant here, so I'll just show a fragment of it, that sets **WEBSITE_RUN_FROM_PACKAGE** app setting to the publicly available package download link:
```
"properties": {
    "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', parameters('functionAppName'))]",
    "siteConfig": {
        "appSettings": [
            {
                "name": "WEBSITE_RUN_FROM_PACKAGE",
                "value": "https://www.nuget.org/api/v2/package/my-nuget-packed-function"
            }
            
            ... more settings go here
        ]
}
```

Yes, **WEBSITE_RUN_FROM_PACKAGE** setting takes a URL (not a local file name), but this provides an extra benefit: you could have your Functions packed, versioned and delivered via a NuGet feed.

Indeed, as of today, there're lots of different ways to deploy your Function App to Azure. It is even possible to have an ARM template, that [deploys your custom Docker image](https://docs.microsoft.com/en-us/azure/azure-functions/functions-infrastructure-as-code#create-a-function-app-2) or [raw source code from your repo](https://docs.microsoft.com/en-us/azure/azure-functions/functions-infrastructure-as-code#customizing-a-deployment). Still, the approach with ARM templates + NuGet feeds might be beneficial in certain cases. E.g. if you want to distribute your Functions in a packed, versioned way, yet still allow them to stay under Consumption Plan (since the *custom Docker container*+*Consumption Plan* combination [is so far not yet supported](https://github.com/Azure/Azure-Functions/issues/1458)).