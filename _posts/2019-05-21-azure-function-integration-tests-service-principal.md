---
title: Integration tests for your B2C RESTful backend with AAD Service Principals.
permalink: /2019/05/21/azure-function-integration-tests-service-principal
---
![teaser]({{ site.url }}/images/aad/teaser1.png)
# Integration tests for your B2C RESTful backend with AAD Service Principals.

Suppose you have a typical SPA (Single-Page Application) written with whatever JS framework you prefer, communicating with a RESTful backend implemented with Azure Functions. Let's say, your solution is B2C (Business-to-Consumer), and therefore you would probably want to authenticate your users via some well-known public authentication providers, e.g. Facebook. And in that case the [**Easy Auth** authentication layer](https://cgillum.tech/2016/05/27/app-service-auth-and-azure-ad-b2c/) would be very handy, abstracting your backend code from authN/authZ-related boilerplate. The only authN/authZ-specific piece in your backend code would then be extracting some basic info about the user - userId etc., and that code [could look like this](https://github.com/scale-tone/WhatIfDemo/blob/master/WhatIfDemo-Functions.Common/Helpers.cs#L65).

OK, so far you've coded and tested your app, deployed it, [configured a CI/CD pipeline](https://docs.microsoft.com/en-us/azure/devops/pipelines/get-started/what-is-azure-pipelines?view=azure-devops), the app is up and running, the users can login to it with their Facebook accounts, and everything is fine. What you would (and should!) want to do next is to write some integration tests. But how to make the tests authenticate against the backend?

You could get yourself some dummy Facebook accounts and automate the authentication step with things like Selenium or PhantomJs. That would take a substantial amount of coding/configuration and it's tricky to make it work on any arbitrary build machine. You could generate your access tokens manually and feed them to your tests, but you'll get tired very soon. You could also make changes to your backend code, specifically to make it work with some other kind of authentication, e.g. secret keys, but that would corrupt the whole idea of integration testing, right?

Instead of all of that you can use a specially designated [Azure Service Principal](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest). Thanks to the way **Easy Auth** is implemented, you can configure it to trust multiple authentication providers, e.g. both Facebook and AAD, and your backend code doesn't need to worry about the details. The userId will still be available to your code via the same [/.auth/me endpoint](https://docs.microsoft.com/en-us/azure/app-service/app-service-web-tutorial-auth-aad#call-api-securely-from-browser-code) (for Facebook it will be Facebook ID, for your test Service Principal it will be Service Principal's ObjectId).

Here is how to configure it.

1. Create a new Service Principal via Azure CLI:
```
  az ad sp create-for-rbac --name <MyIntegrationTestServicePrincipal>
```
and take a note of it's **appId** and **password** from the command output. These will constitute the credentials to be used by your integration test.

2. Get that newly created Service Principal's **ObjectId** (note that it is **not** the same as **appId** returned by the step1):
```
   az ad sp show --id <appId-from-step1>
```
That command outputs some JSON, from which you need to take and memorize the **objectId** value.

3. Create a new AAD application, to be associated with your backend App Service:
```
   az ad app create --display-name <MyAadBackendApp>
```
From the command output memorize the **appId** value, which we will now call **ClientId**.

4. In Azure Portal go to **Azure Active Directory->App Registrations** and find your new AAD app there by display name or by **ClientId**. Then go to **Expose an API** tab and create the **Application ID URI** value for your app, by pressing the **Set** button:

![set-application-id-uri]({{ site.url }}/images/aad/set-application-id-uri.png)

The portal will produce and apply some default value for **Application ID URI**, which you need to memorize as well. Further on we'll refer to this value as **ResourceId**.

5. [Configure AAD authentication](https://docs.microsoft.com/en-us/azure/app-service/configure-authentication-provider-aad) for your backend App Service, in addition to all other types of authentication configured. Choose **Advanced** management mode, specify **ClientId** from step3 as the Client ID and add **ResourceId** from step 4 to the list of **ALLOWED TOKEN AUDIENCES**.

6. Figure out your backend AAD Application's Service Principal's **ObjectId**:
```
   az ad sp list --display-name <MyAadBackendApp>
```
Note that this is a different Service Principal and a different **ObjectId**, it has nothing in common with the one from step2. Also make sure you're taking the **ObjectId** of a Service Principal (not of a Managed Identity, which might as well be output by this command). Another way of obtaining this particular **ObjectId** is to navigate to Azure Active Directory **Enterprise Applications** (not App Registrations!) tap in Azure Portal and find your AAD app by **ClientId** or by name there. Yes, it is very easy to get confused by all these ids, but please bear with me.

7. Now you need to actually grant access rights. That is, to allow your integration test Service Principal (from step1) to access your Function's AAD application (from step3). That needs to be done with this Powershell command:
```
   New-AzureADServiceAppRoleAssignment -Id 00000000-0000-0000-0000-000000000000 -ObjectId <objectId-from-step2> -PrincipalId <objectId-from-step2-again> -ResourceId <objectId-from-step-6>
```
For this cmdlet to be accessible, you first need to install the [AzureAD Powershell Module](https://docs.microsoft.com/en-us/powershell/module/azuread/?view=azureadps-2.0).
A zero GUID is intended, don't be surprised. It just means that we're not using any app roles, and all our tokens are going to be equally powerful. If you do want to have multiple roles in your application, then you'll need to configure them first [by manually editing the manifest](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-add-app-roles-in-azure-ad-apps) and then to specify the **Role Definition Id** instead of a zero GUID.

8. Now finally you can construct the [**Connection String** for AzureServiceTokenProvider](https://docs.microsoft.com/en-us/azure/key-vault/service-to-service-authentication#connection-string-support):
```
   RunAs=App;AppId=<appId-from-step1>;TenantId=<your-AAD-tenantId>;AppKey=<password-from-step1>
```
Save this string as a **AzureServicesAuthConnectionString** environment variable locally or as your Release Pipeline Parameter in Azure DevOps. From now on the integration test should be able to generate an access token: 
```
   var azureServiceTokenProvider = new AzureServiceTokenProvider();
   string accessToken = await azureServiceTokenProvider.GetAccessTokenAsync("<ResourceId>");
```
and call your backend with it. Not directly, of course, but by [exchanging it](https://github.com/scale-tone/WhatIfDemo/blob/master/WhatIfDemo-Functions.IntegrationTest/IntegrationTest.cs#L44) to an **Easy Auth** session token and then submitting this session token via **X-ZUMO-AUTH** HTTP header.

As a wrap-up: indeed, this post appeared to be longer than expected, and those eight simple steps do not look so simple, but I still think the result is worth the effort. A final note: for all of this to work, you Azure Functions backend doesn't need to be .Net-based, you can write in any [supported language](https://docs.microsoft.com/en-us/azure/azure-functions/supported-languages). The same approach with Service Principals could be used for doing service-to-service calls, but a much better idea would be to [utilize Azure Managed Identities for that scenario](https://joonasw.net/view/calling-your-apis-with-aad-msi-using-app-permissions). Also it doesn't necessarity need to be Azure Functions, **Easy Auth** authentication layer is available for anything that's hosted as an Azure App Service.