---
title: Service-to-Service calls with Managed Identities, in Azure and also on your devbox.
permalink: /2019/09/13/service-to-service-managed-identities-devbox
---
![teaser]({{ site.url }}/images/managed-identities/teaser1.png)
# Service-to-Service calls with Managed Identities, in Azure and also on your devbox.

People have already [blogged a lot](https://blog.bredvid.no/protecting-your-asp-net-core-app-with-azure-ad-and-managed-service-identity-78007d7a0774) about Azure Managed Identities, and specifically about [how to secure service-to-service calls with them](https://joonasw.net/view/calling-your-apis-with-aad-msi-using-app-permissions). So here I'll just briefly outline the easiest way to achieve that.

OK, so you have two RESTful services running in Azure - **the-caller** and **the-callee**. And you want to let **the-caller** make authenticated calls against **the-callee**, but don't want to bother yourself with secrets, certificates, Key Vaults etc. 

**The-caller** will typically be your Asp.Net/Asp.Net Core backend hosted as an [Azure App Service, Azure Function App](https://docs.microsoft.com/en-us/azure/app-service/overview-managed-identity) or [Azure Container Instance](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-managed-identity) (could also be language other than C#, but you would need to make handcrafted REST calls for obtaining tokens, rather than using a [library](https://www.nuget.org/packages/Microsoft.Azure.Services.AppAuthentication)). A plain .Net Console App or Windows Service [running on a VM](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/qs-configure-portal-windows-vm) will also do.

**The-callee** can be written in any language, but needs to be hosted either as an App Service or as a Function App.

1. Enable system-assigned Managed Identity for **the-caller**:
    ![image1]({{ site.url }}/images/managed-identities/identity-link.png)

    And take note of it's **Object ID**:
    ![image2]({{ site.url }}/images/managed-identities/identity-object-id.png)

    Let's call it **CallerObjectId** further on.

2. Configure [EasyAuth](https://github.com/cgillum/easyauth/wiki) with AAD in *Express Mode* for **the-callee**:
    ![image3]({{ site.url }}/images/managed-identities/easyauth-link.png)
    
    ![image4]({{ site.url }}/images/managed-identities/easyauth-aad.png)
    
    ![image5]({{ site.url }}/images/managed-identities/easyauth-aad-express-mode.png)

    Note the name of AAD app, that will be created for you as part of *Express Mode* setup. On the screenshot above it is **the-callee-aad-app**.
    NOTE: if your **the-callee** already has an AAD app, no need to recreate it. Also it's perfectly OK to use *Advanced Mode*, just in that case you'll need to create an AAD app and put some more parameters manually.

3. This step is **essential**, otherwise **the-callee** might potentially be left accessible by unauthorized users. Go to **AAD Enterprise Applications**:
    ![image6]({{ site.url }}/images/managed-identities/enterprise-applications-link.png)

    , find **the-callee-aad-app** there by name:
    ![image7]({{ site.url }}/images/managed-identities/enterprise-applications-find-by-name.png)
    
    , open its **Properties** tab and ensure that **User Assignment Required** is set to **Yes** and **Visible to Users** is set to **No**:
    ![image8]({{ site.url }}/images/managed-identities/aad-app-user-assignment-required.png)

    Since **the-callee** should only be callable from another service (and never with a *user-specific* access token), no user should ever be allowed to login to **the-callee-aad-app** with [OAuth2 Implicit Grant flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v1-oauth2-implicit-grant-flow), and that's what we ensure here. 

4. While you're still on the **Properties** tab, take a note of **the-callee-aad-app**'s Object ID. Let's call it **CalleeObjectId**, we'll need it later for making a role assignment.

5. To be even more sure that **the-callee-aad-app** is configured properly, go to **AAD App Registrations**, find **the-callee-aad-app** there by name:
    ![image9]({{ site.url }}/images/managed-identities/app-registrations-link.png)

    and on the **Authentication** tab ensure that neither *Access tokens* nor *ID tokens* are enabled:

    ![image10]({{ site.url }}/images/managed-identities/app-registrations-disable-implicit-grant.png)

6. While **the-callee-aad-app**'s App Registration page is still open, go to **Expose an API** tab and take a note of it's **Application ID URI** value:
    ![image11]({{ site.url }}/images/managed-identities/app-registrations-application-id-uri.png)

    Let's call this value **CalleeResourceId**, and this is what your code will be using when obtaining access tokens. It might happen that your **Application ID URI** is empty - in this case you just need to set it (default value provided by Azure Portal will do).

7. Finally, grant **the-caller** access rights to **the-callee** (yes, [AzureAD Powershell Module](https://docs.microsoft.com/en-us/powershell/module/azuread/?view=azureadps-2.0) is currently the only way of doing this):
```
   New-AzureADServiceAppRoleAssignment -Id 00000000-0000-0000-0000-000000000000 -ObjectId <CallerObjectId from step1> -PrincipalId <CallerObjectId from step1 again> -ResourceId <CalleeObjectId from step4>
```

    The **Id** parameter here is the App Role Definition Id (which you're supposed to previously define [by manually editing AAD app's manifest](https://joonasw.net/view/defining-permissions-and-roles-in-aad)), but if **the-callee** doesn't use app roles (which is quite likely for a typical backend API), then you can save your time and just specify **Guid.Null** (the *Default* role).

    Now, with the following code (being executed on **the-caller**'s instances in Azure):
```
    var tokenProvider = new AzureServiceTokenProvider();
    string accessToken = await tokenProvider.GetAccessTokenAsync("<CalleeResourceId from step6>");
```

    you obtain an access token, which is then accepted by **the-callee**:
    ![image12]({{ site.url }}/images/managed-identities/the-callee-postman.png)

    **AzureServiceTokenProvider** class comes from [Microsoft.Azure.Services.AppAuthentication](https://www.nuget.org/packages/Microsoft.Azure.Services.AppAuthentication) library and does all the token generation magic in C#. In other languages you'll need to [make a local HTTP call](https://docs.microsoft.com/en-us/azure/app-service/overview-managed-identity#using-the-rest-protocol) yourself.



That's all great, you might say, but how do I now debug my stuff locally? Specifically, how do I get access tokens for calling **the-callee** from my devbox?

Fortunately, it is now also possible. The above token generation code can also work locally, with no code changes required! You just need to make a bit more configurations, and here is how.



<span>8.</span> Ensure you're logged in with a proper account into [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) on your devbox:
```
    az account show
```

    
    When running locally, **AzureServiceTokenProvider** tries to reuse your credentials saved by **Azure CLI** for generating tokens, so it is important to be properly logged in.

<span>9.</span> Go to **the-callee-aad-app**'s App Registration page, choose **Expose an API** tab and add **Azure CLI** to the list of *Authorized Client Applications*:
    ![image13]({{ site.url }}/images/managed-identities/add-authorized-client-application.png)

    The **Azure CLI**'s **Client ID** is **04b07795-8ddb-461a-bbee-02f9e1bf7b46**. With this step you're basically declaring: "I, **the-callee**, am now going to accept tokens issued by Azure CLI app for its users, provided that the audience claim in them matches my Resource Id".

<span>10.</span> Now run the token generation code locally and observe an exception:
    ![image14]({{ site.url }}/images/managed-identities/user-not-assigned-role-for-application.png)

    This tells us that not all users are allowed to obtain these tokens, but only whitelisted ones. So, you now need to whitelist yourself (and probably other team members).

<span>11.</span> Find **the-callee-aad-app** in *AAD Enterprise Applications* again, go to **Users and Groups**:
    ![image15]({{ site.url }}/images/managed-identities/enterprise-application-users-groups.png)


    and add some users and/or groups:
    ![image16]({{ site.url }}/images/managed-identities/enterprise-application-add-user.png)


    Give it ~10 minutes to propagate, and if you now re-run the token generation code, the token should be successfully issued and accepted by **the-callee**:
    ![image17]({{ site.url }}/images/managed-identities/the-callee-postman.png)

If you decode this token (with e.g. https://jwt.io), you'll notice that, unlike the token for Managed Identity, this one is *user-specific*, and it has your name in it. And of course, it is important to remember, that steps 8-11 should only be executed for dev environments. Prod (and probably test also) environments should only utilize Managed Identities and never be accessible from devboxes.

The only scenario left untouched here is how to make your integration tests work on an arbitrary build machine. That is also achievable by using *Azure Service Principals*, and it was something [I blogged about a while ago](https://scale-tone.github.io/2019/05/21/azure-function-integration-tests-service-principal).