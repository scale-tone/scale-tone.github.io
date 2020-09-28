---
title: Cognitive Search Static Web Apps Demo.
permalink: /2020/09/28/cognitive-search-static-web-apps-demo
---
![teaser]({{ site.url }}/images/static-web-apps/teaser1.png)
# Cognitive Search Static Web Apps Demo.

While playing with [Azure Static Web Apps](https://docs.microsoft.com/en-us/azure/static-web-apps/) (warning: still in preview as of today) I decided to do something more substantial than a basic React "Hello world" application or [a GitHub template repo](https://github.com/scale-tone/react-ts-basic). It so happened that I was also working extensively with [Azure Cognitive Search](https://azure.microsoft.com/en-us/services/search/) just recently, and so my obvious decision was to implement a simple sample template search UI for it. In fact, [quite](https://docs.microsoft.com/en-us/azure/search/search-create-app-portal) a [bunch](https://github.com/Azure-Samples/azure-search-knowledge-mining/tree/master/02%20-%20Web%20UI%20Template) of [such implementations](https://github.com/microsoft/AzureSearch_JFK_Files#the-jfk-files) exists already on the market, but they are a bit "classic" in terms of technologies being used and a bit heavy in terms of amount of code taken. And also they're hardly hostable as an Azure Static Web App.

So here is my take on this - [Cognitive Search Static Web Apps Demo](https://github.com/scale-tone/cognitive-search-static-web-apps-sample-ui#cognitive-search-static-web-apps-sample). It is implemented as a typical React-based Single-Page App, written in TypeScript, with [MobX](https://mobx.js.org/README.html) used for state management and [Material-UI](https://material-ui.com/) used as a UI framework. And it doesn't have a backend as such. The client code communicates with [Cognitive Search REST API](https://docs.microsoft.com/en-us/azure/search/search-query-overview), but those calls are transparently propagated through an [Azure Functions Proxy](https://docs.microsoft.com/en-us/azure/azure-functions/functions-proxies), the purpose of which is to append the [api-key](https://docs.microsoft.com/en-us/azure/search/search-security-api-keys#create-query-keys) HTTP header to each request. So that the **api-key** value never leaves the server side and is never exposed to the public. One of the nice features of Azure Static Web Apps is that it can have an [Azure Functions-based backend](https://docs.microsoft.com/en-us/azure/static-web-apps/add-api) as part of the same project, in its **/api** folder. And this is where that Azure Functions Proxy resides, it is configured via [this /api/proxies.json](https://github.com/scale-tone/cognitive-search-static-web-apps-sample-ui/blob/master/api/proxies.json) file and is deployed along with all the static content to the same Static Web Apps instance.

The client app implements the user experience known as [faceted search](https://en.wikipedia.org/wiki/Faceted_search) - the user first enters their initial search phrase on the initial landing page and then continues to narrow down search results with multiple filters (facets) on the left sidebar. The list of search results is almost instantly updated once making any changes to the filters. It also features a simple infinite scroll implementation. If the documents in your index have a field with geographical coordinates, the search results will also be shown on a map, or more precisely, on an [Azure Maps](https://azure.microsoft.com/en-us/services/azure-maps/) control. Clicking on a search result produces the *Details* view, where the most useful tab is the *Transcript* tab - it highlights your search phrase in the document's text and allows to navigate across multiple occurences of it:

![transcript]({{ site.url }}/images/static-web-apps/transcript.png)

Last but not least is the matter of authentication/authorization. It is important to remember that by default there will be **no authentication** for your Azure Static Web Apps instance, so everyone will be able to access data in your search index. So it would probably be a good idea to turn authN/authZ rules on as soon as possible, and you can do that by adding the `"allowedRoles": [ "authenticated" ]` line to the pre-existing [/public/routes.json](https://github.com/scale-tone/cognitive-search-static-web-apps-sample-ui/blob/master/public/routes.json) file, so that it looks like this:

```
{
    "routes": [
        {
            "route": "/*",
            "serve": "/index.html",
            "statusCode": 200,
            "allowedRoles": [ "authenticated" ]
        }
    ],
    "platformErrorOverrides": [
        {
            "errorType": "Unauthenticated",
            "statusCode": "302",
            "serve": "/.auth/login/aad"
        }
    ]
}
```

With that change, every user trying to access your app will be redirected to AAD login page (can also be GitHub, Facebook, Google or Twitter, if you change the `"serve"` field below accordingly). Since then all your users will be required to authenticate, but it still means that anybody can access your app. To restrict the access rights further you'll need to create a custom role, [assign it to specific users](https://docs.microsoft.com/en-us/azure/static-web-apps/authentication-authorization#role-management) and then replace `"authenticated"` with that custom role name in the list of **allowedRoles**.

The project is intended to be connectable to any Cognitive Search index, so long as you fork [the repo](https://github.com/scale-tone/cognitive-search-static-web-apps-sample-ui) and then configure your copy with proper [config settings](https://github.com/scale-tone/cognitive-search-static-web-apps-sample-ui#config-settings). I've also deployed and configured the [live demo](https://lively-sand-033e9ec03.azurestaticapps.net/) of it, which is currently pointed to [this freely available `hotels` demo index](https://docs.microsoft.com/en-us/samples/azure-samples/azure-search-sample-data/azure-search-sample-data/). It would make sense to use that one as a playground and then proceed with building and connecting your own index. 

A lot of useful functionality provided by Cognitive Search is still missing in this demo, but I hope it can already serve as a good starting point for both Cognitive Search and Static Web Apps. Please, enjoy and don't hesitate to make your own contributions to this codebase.