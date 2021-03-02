---
title: "Event-Driven KeyVault Secrets Rotation Management"
slug: event-driven-keyvault-secrets-rotation-management
description: "This post shows how to implement e2e workflow process using Azure Key Vault, Azure Event Grid, Azure Logic Apps and Azure Functions, to manage secrets rotation management when a new secret version is added."
date: "2021-02-24"
author: Justin-Yoo
tags:
- azure-keyvault
- azure-eventgrid
- azure-logic-apps
- azure-functions
- azure-sdk
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/02/event-driven-keyvault-secrets-rotation-management-00.png
fullscreen: true
---


In my [previous post][post prev 1], I discussed how all [secrets][az kv secrets] in [Azure Key Vault][az kv] could automatically manage their versions to get disabled. While that approach was surely useful, it sometimes seems overkilling to iterate against all secrets at once. What if you can manage only a certain secret when the secret gets a new version updated? That could be more cost-effective. As an Azure Key Vault instance publishes events through [Azure EventGrid][az evtgrd], by capturing this event, you can [manage version rotation][az kv secrets rotation].

* [KeyVault Secrets Rotation Management][post prev 1]
* ***Event-Driven KeyVault Secrets Rotation Management***

Throughout this post, I'm going to discuss how to handle events published through [Azure EventGrid][az evtgrd] and manage secret rotations using [Azure Logic Apps][az logapp] and [Azure Functions][az fncapp] when a new secret version is created.

> You can download the sample code from this [GitHub repository][gh sample].


## Events from Azure Key Vault ##

Azure Key Vault publishes events to [Azure EventGrid][az kv evtgrd]. Whenever a new secret version is added, it always [raises an event][az kv evtgrd type]. Therefore, processing this event doesn't have to iterate all secrets but focuses on the specific secret, making our lives easier. Here's the high-level end-to-end workflow architecture using [Azure Key Vault][az kv], [Azure EventGrid][az evtgrd], [Azure Logic Apps][az logapp] and [Azure Functions][az fncapp].

![Overall E2E Process Architecture][image-01]

Like I mentioned in my [previous post][post prev 2], using [Azure Logic Apps][az logapp] as an event handler doesn't require the [event delivery authentication][az evtgrd delivery auth]. But if you prefer explicit authentication, please refer to my [another blog post][post prev 3].

There are two ways to integrate Azure Key Vault with Azure Logic App as an event handler. One uses the EventGrid trigger through the [connector][az logapp connectors evtgrd], and the other uses the HTTP trigger like a regular HTTP API call. While the former generates a dependency on the [connector][az logapp connectors], the latter works both instances independently, which is my preferred approach.

First of all, create an Azure Logic App instance and add the [HTTP trigger][az logapp connectors request].

![Logic Apps HTTP Trigger][image-02]

Once save the Logic App workflow, you will get the endpoint URL, which will be used as the event handler webhook. Go to the Azure Key Vault instance's Events blade and click the `+ Event Subscription` button.

![Event Subscription Button][image-03]

You will be asked to create an EventGrid subscription instance. Enter `Event Subscription Details Name`, `Event Schema`, `System Topic Name`, `Event Type`, `Endpoint Type` and `Endpoint URL`.

![Event Subscription Details][image-04]

* In the Event Subscription Details session, choose [Cloud Event Schema v1.0][ce spec http] because it's the [standard spec][ce spec] of [CNCF][cncf] and it's convenient for heterogeneous systems integration.
* Enter the Event Grid Topic name to the `System Topic Name` field.
* Choose only the `Secret New Version Created` event in the `Filter to Event Types` dropdown.
* Choose Webhook and enter the endpoint URL copied from the Logic App HTTP trigger.

You've completed the very basic pipeline between Azure Key Vault, Azure EventGrid and Azure Logic Apps to handle events. If you create a new version of a particular secret, it generates an event captured by the Logic App instance. Confirm that the `Microsoft.KeyVault.SecretNewVersionCreated` event type has been captured.

![Event Captured by Logic App][image-05]

The actual event data as a JSON payload looks like this:

![Event Data Payload in Logic App][image-06]

There is the attribute called `ObjectName` in the `data` attribute, which is the secret name. You need to send this value to Azure Functions to process the secret version rotation management. Let's implement the function logic.


## Version Rotation Management against Specific Secret via Azure Functions ##

There are not many differences from my [previous post][post prev 1]. However, this implementation time will become simpler because it doesn't have to iterate all the secrets at once but look after a specific one. First of all create a new [HTTP Trigger][az fncapp trigger http].

https://gist.github.com/justinyoo/948385359cc739a48ad5afdf07db932e?file=01-func-new-http-trigger.sh

A new HTTP trigger has been generated with the default template. Now, update the `HttpTrigger` binding settings. Remove the `GET` method and put the routing URL to `secrets/{name}/disable/{count:int?}` (line #5). Notice that the routing URL contains placeholders like `{name}` and `{count:int?}`, which are substituted with parameters of `string name` and `int? count` respectively (line #6).

https://gist.github.com/justinyoo/948385359cc739a48ad5afdf07db932e?file=02-disable-secret-http-trigger-01.cs&highlights=5,6

Get the two values from the environment variables. One is the endpoint URL to the Azure Key Vault instance, and the other is the tenant ID where the Key Vault instance is hosted.

https://gist.github.com/justinyoo/948385359cc739a48ad5afdf07db932e?file=02-disable-secret-http-trigger-02.cs

Next, instantiate the `SecretClient` object that can access the Key Vault instance. While instantiating, give the authentication options with the `DefaultAzureCredentialOptions` object. If your log-in account is bound with multiple tenants, you should explicitly specify the tenant ID; otherwise, you will get the authentication error (line #4-6).

https://gist.github.com/justinyoo/948385359cc739a48ad5afdf07db932e?file=02-disable-secret-http-trigger-03.cs&highlights=4-6

As you already know the secret name, populate all the versions of the given secrets. Of course, you don't need inactive versions. Therefore, use the `WhereAwait` clause to filter them out (line #5). Additionally, use the `OrderByDescendingAwait` clause to sort all the active versions in the reverse-chronological order (line #6).

https://gist.github.com/justinyoo/948385359cc739a48ad5afdf07db932e?file=02-disable-secret-http-trigger-04.cs&highlights=5,6

If there is no version enabled, end the function by returning the `AcceptedResult` instance.

https://gist.github.com/justinyoo/948385359cc739a48ad5afdf07db932e?file=02-disable-secret-http-trigger-05.cs

As you need at least two versions enabled for rotation if there is no `count` value given, set the value to `2` as the default.

https://gist.github.com/justinyoo/948385359cc739a48ad5afdf07db932e?file=02-disable-secret-http-trigger-06.cs

If the number of secret versions enabled is less than the `count` value, complete the processing and return the `AcceptedResult` instance.

https://gist.github.com/justinyoo/948385359cc739a48ad5afdf07db932e?file=02-disable-secret-http-trigger-07.cs

Let's disable the remaining versions. Skip as many as the `count` value of the versions (line #2). Set the `Enabled` value to `false` (line #7), the update them (line #9).

https://gist.github.com/justinyoo/948385359cc739a48ad5afdf07db932e?file=02-disable-secret-http-trigger-08.cs&highlights=2,7,9

Finally, return the processed result as a response.

https://gist.github.com/justinyoo/948385359cc739a48ad5afdf07db932e?file=02-disable-secret-http-trigger-09.cs

The implementation of the Azure Functions side is over. Let's integrate it with [Azure Logic Apps][az logapp].


## Integration of Azure Logic Apps with Azure Functions ##

Add both HTTP action and Response action to the Logic App instance previously generated. Make sure that you call the Azure Functions app through the HTTP action, with the `ObjectName` value and `2` as the routing parameters.

![Additional Actions to Logic Apps][image-07]

Now, you've got the integration workflow completed from Azure Key Vault to Azure Functions via Azure EventGrid and Logic Apps. Let's run the workflow.


## End-to-End Test &ndash; Adding New Secret Version to Azure Key Vault ##

In order to run the integrated workflow, you need to create a new version of the Azure Key Vault Secrets.

![List of Azure Key Vault Secrets][image-08]

Add a new version of the secret.

![Adding a New Version of Secret][image-09]

You will see the new version added.

![Result of the New Version of Secret Added][image-10]

When a new secret version is added, it publishes an event to EventGrid, and the Logic App captures the event. Can you confirm the `ObjectName` value and the secret version are the same as the one on the Azure Key Vault instance?

![Logic App Run Result][image-11]

Once you complete the end-to-end integration workflow, you will be able to see that all versions except the latest two have been disabled.

![Secret Versions Disabled][image-12]

---

So far, we've implemented a new logic that captures an event published when a new [secret version][az kv secrets] is added to [Azure Key Vault][az kv] instance, and process the rotation management against the specific secret, using [Azure EventGrid][az evtgrd], [Azure Logic Apps][az logapp] and [Azure Functions][az fncapp]. It would be handy if you have a similar use case and implement this sort of event-driven workflow process.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/02/event-driven-keyvault-secrets-rotation-management-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/02/event-driven-keyvault-secrets-rotation-management-02-en.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/02/event-driven-keyvault-secrets-rotation-management-03-en.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/02/event-driven-keyvault-secrets-rotation-management-04-en.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/02/event-driven-keyvault-secrets-rotation-management-05-en.png
[image-06]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/02/event-driven-keyvault-secrets-rotation-management-06-en.png
[image-07]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/02/event-driven-keyvault-secrets-rotation-management-07-en.png
[image-08]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/02/event-driven-keyvault-secrets-rotation-management-08-en.png
[image-09]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/02/event-driven-keyvault-secrets-rotation-management-09-en.png
[image-10]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/02/event-driven-keyvault-secrets-rotation-management-10-en.png
[image-11]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/02/event-driven-keyvault-secrets-rotation-management-11-en.png
[image-12]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/02/event-driven-keyvault-secrets-rotation-management-12-en.png

[post prev 1]: /2021/02/17/keyvault-secrets-rotation-management/
[post prev 2]: /2021/01/27/websub-to-eventgrid-via-cloudevents-and-beyond/
[post prev 3]: /2021/01/13/dealing-cloudevents-with-azure-functions-for-azure-eventgrid/

[gh sample]: https://github.com/devkimchi/KeyVault-Reference-Sample/tree/2021-02-24

[az logapp]: https://docs.microsoft.com/azure/logic-apps/logic-apps-overview?WT.mc_id=dotnet-17246-juyoo
[az logapp connectors]: https://docs.microsoft.com/connectors/connectors?WT.mc_id=dotnet-17246-juyoo
[az logapp connectors request]: https://docs.microsoft.com/azure/connectors/connectors-native-reqres?WT.mc_id=dotnet-17246-juyoo
[az logapp connectors evtgrd]: https://docs.microsoft.com/connectors/azureeventgrid/?WT.mc_id=dotnet-17246-juyoo

[az fncapp]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-17246-juyoo
[az fncapp trigger http]: https://docs.microsoft.com/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=csharp&WT.mc_id=dotnet-17246-juyoo

[az kv]: https://docs.microsoft.com/azure/key-vault/general/overview?WT.mc_id=dotnet-17246-juyoo
[az kv secrets]: https://docs.microsoft.com/azure/key-vault/secrets/about-secrets?WT.mc_id=dotnet-17246-juyoo
[az kv secrets rotation]: https://docs.microsoft.com/azure/app-service/app-service-key-vault-references?WT.mc_id=dotnet-17246-juyoo#rotation
[az kv evtgrd]: https://docs.microsoft.com/azure/key-vault/general/event-grid-overview?WT.mc_id=dotnet-17246-juyoo
[az kv evtgrd type]: https://docs.microsoft.com/azure/event-grid/event-schema-key-vault?tabs=cloud-event-schema&WT.mc_id=dotnet-17246-juyoo

[az evtgrd]: https://docs.microsoft.com/azure/event-grid/overview?WT.mc_id=dotnet-17246-juyoo
[az evtgrd delivery auth]: https://docs.microsoft.com/azure/event-grid/security-authentication?WT.mc_id=dotnet-17246-juyoo

[cncf]: https://cncf.io/

[ce]: https://cloudevents.io/
[ce spec]: https://github.com/cloudevents/spec/tree/v1.0
[ce spec http]: https://github.com/cloudevents/spec/blob/v1.0/http-protocol-binding.md
