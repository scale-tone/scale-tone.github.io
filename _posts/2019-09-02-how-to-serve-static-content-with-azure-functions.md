---
title: How to serve static content with Azure Functions.
permalink: /2019/09/02/how-to-serve-static-content-with-azure-functions
---
![teaser]({{ site.url }}/images/functions/teaser1.png)
# How to serve static content with Azure Functions.


## **Easy!**

C#:
<script src="https://gist.github.com/scale-tone/e19d4edab0231aff97a55243d811ad12.js"></script>

TypeScript:
<script src="https://gist.github.com/scale-tone/e311e6cc52fcc7d79e1a06d4e6cb6dd7.js"></script>

The above file mappings are relevant for React-based web apps, but you can easily change it to match e.g. Angular's build output file structure. The statics themselves you just need to copy into your Azure Function's project folder. If it is C#, then also make them be copied to the build output via your CSPROJ-file:
<script src="https://gist.github.com/scale-tone/7d4e29381aed34b8536db04664992c2e.js"></script>

Yes, basically, we're indeed re-inventing a web server here. But there're **benefits**.

**Benefit #1**. No need to search for a separate hosting solution, e.g. [static website hosting in Azure Storage](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-static-website) or another Azure App Service instance.

**Benefit #2**. Since your statics are now under the same base URL as your REST backend - no need to worry about CORS!

**Benefit #3**. By configuring [EasyAuth](https://docs.microsoft.com/en-us/azure/app-service/overview-authentication-authorization#authentication-flow) with [server-directed login flow](https://github.com/cgillum/easyauth/wiki/Login#server-directed-login) you get basic authN/authZ functionality **for free**, without any client-side SDKs or extra JavaScript code required. Configure the AAD authentication provider for your Azure Function App with **Express** setup: 
![image1]({{ site.url }}/images/aad/express-setup.png)

and set **Action to take when request is not authenticated** to **Log in with Azure Active Directory**: 
![image2]({{ site.url }}/images/aad/login-with-aad.png)

Since then any unauthenticated request (including requests for your static files) will be redirected to AAD login/consent page: 
![image3]({{ site.url }}/images/aad/aad-login.png)

Upon successful authentication it will get an **AppServiceAuthSession** cookie, (since, again, the base URL is the same) each subsequent request made by your client code will inherit it and get authenticated as well. The only thing that is worth being handled explicitly on your client side is auth-related errors when the cookie gets expired. You can just refresh the page in that case:
<script src="https://gist.github.com/scale-tone/c2932ee077e7c12938225c81c5a00511.js"></script>

WARNING: by default, the AAD app that is created during EasyAuth Express setup allows **anyone** in your tenant to login and get access! To reduce the list of allowed users you need to open that AAD app in **Enterprise Applications**, set **User assignment required** to **Yes**: 
![image4]({{ site.url }}/images/aad/user-assignment-required.png)

and then add users and/or groups explicitly via the **Users and groups** page:
![image5]({{ site.url }}/images/aad/users-and-groups.png)

Or implement [custom claim and/or role validation](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=csharp#working-with-client-identities) in your Azure Function code.

UPD1: the above examples will make your statics available via a URL like *https://[your-host-name]/api/ui*, but you could also easily make them served from the root (*https://[your-host-name]*). Just set the **routePrefix** setting to a blank string in your **host.json**:
```
{
    "version": "2.0",
    "extensions": {
        "http": {
            "routePrefix": ""
        }
    }
}
```
Though this change will also overshadow all your HTTP-triggered API methods (if you have any), so you will also need to explicitly define routings for those.

UPD2: and for pure TypeScript-based projects I now have this ready-to-use scaffolding in form of a GitHub template repo - [durable-mvc-starter](https://github.com/scale-tone/durable-mvc-starter). It includes both server-side (Azure Functions) and client-side (React+MobX) projects, they are both automatically compiled together, and server side is pre-configured to serve client statics, thus leveraging the above mentioned benefits.
