---
title: "AuthN-ing Blazor WASM with Azure AD B2C"
slug: authn-ing-blazor-wasm-with-azure-ad-b2c
description: "Throughout this post, I'm going to walk through how to integrate Azure AD B2C with Blazor WASM (standalone) app."
date: "2022-09-23"
author: Justin-Yoo
tags:
- dotnet
- blazor-wasm
- azure-ad-b2c
- authn
cover: /2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-00-en.png
fullscreen: true
---

There are many ways to implement authentication and authorisation onto the [Blazor WebAssembly (WASM)][blazor wasm] app. If you publish it to [Azure Static Web Apps (ASWA)][az swa], you can use the built-in authN feature dealt with in my [previous post][post 1]. What if you're publishing your Blazor WASM app onto GitHub Pages? There's no built-in authN feature in this case. However, [Azure AD B2C][az ad b2c] offers a straightforward way for it. You can see literally nothing to write codes to make this happen except a few configurations. Throughout this post, I'm going to discuss how to integrate Azure AD B2C with a standalone type Blazor WASM app.

> You can download a sample application from this [GitHub repository][gh sample].


## Azure AD vs Azure AD B2C ##

As the naming implies, both [Azure AD][az ad] and [Azure AD B2C][az ad b2c] are similar to each other. In terms of the authentication and authorisation service provider, both are basically the same as each other.

What are some differences, then? Azure AD does granular controls for internal (organisation-wide) resources like SharePoint, Teams, Dynamics365, etc. Azure AD sets permissions on each resource and account. On the other hand, Azure AD B2C provides the same authN-ing feature, but it's for specific applications. As Azure AD B2C works independently from Azure AD, it's NOT for organisation-wide purposes unless it's configured in that way.


## Configuring Azure AD B2C ##

Create an Azure AD B2C instance from Azure Portal. You can give any name for it, but let's say `fitabilitydevkr` for now. Then, you'll get the Azure AD B2C instance like below.

![Azure AD B2C][image-01]

Click the "Open B2C Tenant" link to access the Azure AD B2C admin page. On the admin page, click the "App registrations" menu on the left. Then click the "New registration" button at the top.

![Azure AD B2C - New app][image-02]

Enter the details like below:

* Name: Name of the app. Say, `my-fitability-app`.
* Supported account types: Choose the option "Accounts in any identity provider or organisational directory".
* Redirect URI (Recommended): Select "Single-page Application (SPA)", then enter `https://localhost/authentication/login-callback`.
* Permissions: Tick "Grant admin consent to openid and offline_access permissions"

![Azure AD B2C - App registration][image-03]

Once registered, double-check the following details:

* Single-page Application &ndash; Redirect URIs
* Implicit grant and hybrid flows &ndash; Access tokens, ID tokens
* Supported account types &ndash; Accounts in any identity provider or organisational directory

![Azure AD B2C - Authentication][image-04]

You can now see the new app registered. Note the application (client) ID.

![Azure AD B2C - Client ID][image-05]

Let's define the log-in process. On the Azure AD B2C page, click the "User flows" menu on the left, then the "New user flow" menu at the top.

![Azure AD B2C - User flow][image-06]

There are many different user flow types. This time, choose the "Sign up and sign in" flow. Then, select "Recommended".

![Azure AD B2C - Sign up and sign in #1][image-07]

Enter the following values, and click "Create".

* Name: `SignUpSignIn`
* Identity providers: Email Signup

> You can add as many identity providers as you like. But in this post, you can only choose the email sign-up option by default.

![Azure AD B2C - Sign up and sign in #2][image-08]

Tick more options if you want to include more details in the token.

![Azure AD B2C - Sign up and sign in #3][image-09]

You now complete Azure AD B2C configurations.


## Building Blazor WASM App ##

To implement the authN feature with [Azure AD B2C][az ad b2c] on a [Blazor WASM][blazor wasm] app, you first need to create a Blazor WASM project. At the time of this writing, the latest version of [Visual Studio][vs 2022] is `17.3.4`. So, unfortunately, it doesn't create the Blazor WASM project with Azure AD B2C.

![Blazor WASM in VS2022][image-10]

Therefore, you SHOULD use dotnet CLI for it. Enter the command below. You might be noticed the following options:

* `--framework net6.0`: Without giving the framework option explicitly, it uses the latest version of the .NET framework. It explicitly declares the `net6.0` value to fix the framework version.
* `--hosted false`: If you want to create a standalone Blazor WASM app, use this option. If you want a hosted Blazor WASM app, give this option with `true`.
* `--auth IndividualB2C`: Choose this option to use Azure AD B2C.
* `--aad-b2c-instance "https://<TENANT_NAME>.b2clogin.com/"`: It's the login URL of the Azure AD B2C instance. In this post, let's use the tenant name `fitabilitydevkr`.
* `--domain "<TENANT_NAME>.onmicrosoft.com"`: It's the domain URL of the Azure AD B2C instance. In this post, let's use the tenant name `fitabilitydevkr`.
* `--client-id`: The client ID value from the app created in the previous section.
* `--susi-policy-id "B2C_1_SignUpSignIn"`: The policy name for authN. Use the policy name that you just created. In this post, use `SignUpSignIn`.

```powershell
dotnet new blazorwasm \
    --output "MyBlazorWasmApp" \
    --framework net6.0 \
    --hosted false \
    --auth IndividualB2C \
    --aad-b2c-instance "https://<TENANT_NAME>.b2clogin.com/" \
    --domain "<TENANT_NAME>.onmicrosoft.com" \
    --client-id "<CLIENT_ID>" \
    --susi-policy-id "B2C_1_SignUpSignIn"
```

Now, you've got Azure AD B2C enabled on the Blazor WASM app. Let's configure the app. Open `Program.cs` and add the `AddMsalAuthentication(...)` method.

```csharp
builder.Services.AddScoped(sp => new HttpClient
{
    BaseAddress = new Uri(builder.HostEnvironment.BaseAddress)
});

// ⬇️⬇️⬇️ Add these lines below ⬇️⬇️⬇️
builder.Services.AddMsalAuthentication(options =>
{
    options.ProviderOptions.DefaultAccessTokenScopes.Add("openid");
    options.ProviderOptions.DefaultAccessTokenScopes.Add("offline_access");

    builder.Configuration.Bind("AzureAdB2C", options.ProviderOptions.Authentication);
});
/// ⬆️⬆️⬆️ Add these lines above ⬆️⬆️⬆️

await builder.Build().RunAsync();
```

Then, update `appsettings.json` under the `wwwroot` directory.

```json
{
  "AzureAdB2C": {
    "Authority": "https://<TENANT_NAME>.b2clogin.com/<TENANT_NAME>.onmicrosoft.com/B2C_1_SignUpSignIn",
    "ClientId": "<CLIENT_ID>",
    "ValidateAuthority": false
  }
}
```

All the configurations on the Blazor WASM app side are done! Build and run the app on your local machine.

![Blazor WASM landing page][image-11]

You will see the "Log in" button at the top right corner. Click it to see the log-in pop-up window.

![Blazor WASM login page][image-12]

Add your email address and password to log-in, or register a new account by clicking the "Sign up now" button. Once you get logged in, you will see the screen like below:

![Blazor WASM logged in][image-13]

Can you see your username?

---

So far, I've walked through how to add the Azure AD B2C authN feature to the standalone type Blazor WASM app. With minimum code change, you can quickly implement the authN feature. In the next post, I'll give a try to add a social media log-in feature to Azure AD B2C.


## Want to know more about Blazor? ##

It's great if you visit those sites for more Blazor stuff.

* [Blazor][blazor]
* [Blazor Tutorials][blazor tutorial]
* [Blazor Learn][blazor learn]


## Want to know more about Azure AD B2C with Blazor? ##

* [Standalone type Blazor WASM with Azure AD B2C][blazor wasm standalone az ad b2c]
* [Hosted Blazor WASM with Azure AD B2C][blazor wasm hosted az ad b2c]
* [More security scenarios for Blazor WASM][blazor wasm additional scenarios]
* [Azure AD B2C user flow policy][az ad b2c userflow]


[image-01]: /2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-01-en.png
[image-02]: /2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-02-en.png
[image-03]: /2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-03-en.png
[image-04]: /2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-04-en.png
[image-05]: /2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-05-en.png
[image-06]: /2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-06-en.png
[image-07]: /2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-07-en.png
[image-08]: /2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-08-en.png
[image-09]: /2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-09-en.png
[image-10]: /2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-10-en.png
[image-11]: /2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-11-en.png
[image-12]: /2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-12-en.png
[image-13]: /2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-13-en.png


[post 1]: /2021/09/15/accessing-msgraph-from-blazor-wasm-running-on-aswa/

[gh sample]: https://github.com/fitability/fitability-app

[vs 2022]: https://visualstudio.microsoft.com/vs/?WT.mc_id=dotnet-77749-juyoo

[blazor]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor?WT.mc_id=dotnet-77749-juyoo
[blazor tutorial]: https://dotnet.microsoft.com/learn/aspnet/blazor-tutorial/intro?WT.mc_id=dotnet-77749-juyoo
[blazor learn]: https://learn.microsoft.com/training/paths/build-web-apps-with-blazor/?WT.mc_id=dotnet-77749-juyoo
[blazor wasm]: https://learn.microsoft.com/aspnet/core/blazor/?WT.mc_id=dotnet-77749-juyoo#blazor-webassembly
[blazor wasm standalone az ad b2c]: https://learn.microsoft.com/aspnet/core/blazor/security/webassembly/standalone-with-azure-active-directory-b2c?WT.mc_id=dotnet-77749-juyoo
[blazor wasm hosted az ad b2c]: https://learn.microsoft.com/aspnet/core/blazor/security/webassembly/hosted-with-azure-active-directory-b2c?WT.mc_id=dotnet-77749-juyoo
[blazor wasm additional scenarios]: https://learn.microsoft.com/aspnet/core/blazor/security/webassembly/additional-scenarios?WT.mc_id=dotnet-77749-juyoo

[az swa]: https://learn.microsoft.com/azure/static-web-apps/overview?WT.mc_id=dotnet-77749-juyoo
[az ad]: https://learn.microsoft.com/azure/active-directory/fundamentals/active-directory-whatis?WT.mc_id=dotnet-77749-juyoo
[az ad b2c]: https://learn.microsoft.com/azure/active-directory-b2c/overview?WT.mc_id=dotnet-77749-juyoo
[az ad b2c userflow]: https://learn.microsoft.com/azure/active-directory-b2c/tutorial-create-user-flows?pivots=b2c-user-flow&WT.mc_id=dotnet-77749-juyoo
