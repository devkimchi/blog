---
title: "KeyVault Secrets Rotation Management"
slug: keyvault-secrets-rotation-management
description: "This post shows how to automatically disable old versions of each secret in Azure Key Vault at once, using Azure Functions and new Azure SDKs."
date: "2021-02-17"
author: Justin-Yoo
tags:
- azure-keyvault
- azure-functions
- azure-sdk
- secret-management
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/02/keyvault-secrets-rotation-management-00.png
fullscreen: true
---

There was an [announcement][az kv announcement] that you could refer to [Azure Key Vault][az kv] secrets from either [Azure App Service][az appsvc] or [Azure Functions][az fncapp], without having to put their versions explicitly. Therefore, the second approach mentioned in my [previous post][post prev] has become now the most effective way to access [Azure Key Vault Secrets][az kv secrets].

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=01-keyvault-reference.txt

With this approach, the reference always returns the latest version of the secret. Make sure that, when a newer version of the secret is created, it takes up to one day to get synced. Therefore, if your new version of the secret is less than one day old, you should consider the [rotation][az kv secrets rotation]. For the rotation, the ideal number of versions of each secret could be two. If there are more than two versions in one secret, it's better to disable them all the older ones for the sake of security.

* ***KeyVault Secrets Rotation Management***
* [Event-Driven KeyVault Secrets Rotation Management][post next]


As there's no maximum number of secrets defined in [Azure Key Vault][az kv], sometimes there are too many secrets stored in one Key Vault instance. In this case, finding old versions of secrets and disable them by hand should consider automation; otherwise, it needs too many hands. This sort of automation can be done by [Azure Functions][az fncapp] with the Azure Key Vault SDK. Let me show how to do so in this post.

> You can find the sample code used in this post at this [GitHub repository][gh sample].


## Azure Key Vault SDK ##

There are currently two SDKs taking care of Azure Key Vault.

* [Microsoft.Azure.KeyVault][nuget sdk kv old]
* [Azure.Security.KeyVault.Secrets][nuget sdk kv new]

As the first one has been deprecated, you should use the second one. In addition to that, use [Azure.Identity][nuget sdk identity] SDK for authentication and authorisation. Once you create a new Azure Functions project, run the following commands to install these two NuGet packages.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=02-add-nuget-packages.sh

The Key Vault package uses the `IAsyncEnumerable` interface. Therefore, also install this [System.Linq.Async][nuget linq async] package.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=03-add-nuget-packages.sh

> **NOTE**: As of this writing, Azure Functions doesn't support .NET 5 yet. Therefore avoid installing 5.0.0 version of the `System.Linq.Async` package.

We've got all the libraries necessary. Let's build a Functions app.


## Building Functions Code to Disable Old Versions of Each Secret ##

Run the following command that creates a new [HTTP Trigger][az fncapp trigger http] function.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=04-add-new-httptrigger.sh

You've got the basic function endpoint with default settings. Change the `HttpTrigger` binding values. Leave the `POST` method only and enter the routing URL of `secrets/all/disable` (line #5).

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-01.cs&highlights=5

Populate two values from the environment variables. One is the URL of the Key Vault instance, and the other is the tenant ID where the Key Vault instance is currently hosted.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-02.cs

Then, create the `SecretClient` that accesses the Key Vault instance. While instantiating the client, you should provide the `DefaultAzureCredentialOptions` instance as well. If the account logged into Azure is able to access multiple tenants, without explicitly providing the tenant ID, it throws the [authentication error][nuget sdk identity error] (line #4-6).

> It happens more frequently on your local machine than on Azure.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-03.cs&highlights=4-6

Once logged in, get all secrets, iterate them and process each one of them. First things first, let's get all the secrets (line #2-4).

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-04.cs&highlights=2-4

Now, iterate all the secrets and process them. But we don't need all the versions of each secret but need only `Enabled` versions. Therefore use `WhereAwait` for filtering out (line #7). Then, sort them in the reverse-chronological order by using `OrderByDescendingAwait` (line #8). Now, you'll have got the latest version at first.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-05.cs&highlights=7,8

If there is no active version in the secret, stop processing and continue to the next one.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-06.cs

If there is only one active version in the secret, stop processing and continue to the next.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-07.cs

If the latest version of the secret is less than one day old, the rotation is still necessary. Therefore, stop processing and continue to the next one.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-08.cs

Now, the secret has more than two versions and needs to disable the old ones. Skip the first (latest) one process the next one (line #2), set the `Enabled` to `false` (line #6), and update it (line #8).

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-09.cs&highlights=2,6,8

And finally, store the processed result into the response object, and return it.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-10.cs

You've got the logic ready! Run the Function app, and you will see that all the secrets have been updated with the desired status. Suppose you change the trigger from [HTTP][az fncapp trigger http] to [Timer][az fncapp trigger timer], or integrate the current HTTP trigger with [Azure Logic App][az logapp] with scheduling. In that case, you won't have to worry about older versions of each secret to being disabled.

---

So far, we've walked through how an Azure Functions app can manage older versions of each [secret of Azure Key Vault][az kv secrets] while [Azure App Service][az appsvc] and [Azure Functions][az fncapp] are referencing the ones in [Azure Key Vault][az kv]. I hope that this sort of implementation can reduce the amount of management overhead. In the [next post][post next], let's go further to make use of the event published by Azure Key Vault.


[post prev]: /2020/04/30/3-ways-referencing-azure-key-vault-from-azure-functions/
[post next]: /2021/02/24/event-driven-keyvault-secrets-rotation-management/

[gh sample]: https://github.com/devkimchi/KeyVault-Reference-Sample/tree/2021-02-17

[az logapp]: https://docs.microsoft.com/azure/logic-apps/logic-apps-overview?WT.mc_id=dotnet-16807-juyoo

[az appsvc]: https://docs.microsoft.com/azure/app-service/?WT.mc_id=dotnet-16807-juyoo

[az fncapp]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-16807-juyoo
[az fncapp trigger http]: https://docs.microsoft.com/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=csharp&WT.mc_id=dotnet-16807-juyoo
[az fncapp trigger timer]: https://docs.microsoft.com/azure/azure-functions/functions-bindings-timer?tabs=csharp&WT.mc_id=dotnet-16807-juyoo

[az kv]: https://docs.microsoft.com/azure/key-vault/general/overview?WT.mc_id=dotnet-16807-juyoo
[az kv announcement]: https://azure.microsoft.com/updates/versions-no-longer-required-for-key-vault-references-in-app-service-and-azure-functions/?WT.mc_id=dotnet-16807-juyoo
[az kv secrets]: https://docs.microsoft.com/azure/key-vault/secrets/about-secrets?WT.mc_id=dotnet-16807-juyoo
[az kv secrets rotation]: https://docs.microsoft.com/azure/app-service/app-service-key-vault-references?WT.mc_id=dotnet-16807-juyoo#rotation

[nuget sdk kv old]: https://www.nuget.org/packages/Microsoft.Azure.KeyVault/
[nuget sdk kv new]: https://www.nuget.org/packages/Azure.Security.KeyVault.Secrets/
[nuget linq async]: https://www.nuget.org/packages/System.Linq.Async/
[nuget sdk identity]: https://www.nuget.org/packages/Azure.Identity/
[nuget sdk identity error]: https://github.com/Azure/azure-sdk-for-net/issues/11559#issuecomment-620233531
