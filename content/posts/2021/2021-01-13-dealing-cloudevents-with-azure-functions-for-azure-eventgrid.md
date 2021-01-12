---
title: "Dealing CloudEvents with Azure Functions for Azure EventGrid"
slug: dealing-cloudevents-with-azure-functions-for-azure-eventgrid
description: "This post discusses how to send event data to and receive it from Azure EventGrid, in the format of CloudEvents, which is not supported by Azure EventGrid binding extension yet."
date: "2021-01-13"
author: Justin-Yoo
tags:
- azure-functions
- azure-eventgrid
- cloudevents
- azure-sdk
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/01/dealing-cloudevents-with-azure-functions-for-azure-eventgrid-00.png
fullscreen: true
---

[Azure Functions][az fncapp] provides the built-in [binding extensions feature][az fncapp binding evtgrd] for [Azure EventGrid][az evtgrd], which makes event data easier to publish and consume. But, the binding extension doesn't support the [CloudEvents][ce] format yet because it uses the [current version of SDK][nuget evtgrd legacy] that doesn't support the CloudEvents schema yet. I suspect the binding extension will be available for CloudEvents when a [new version of SDK][nuget evtgrd new] becomes GA. Therefore, it requires a few extra coding efforts to send and receive CloudEvents data via Azure Functions app for now. Throughout this post, I will discuss how to make it.

> I used [.NET SDK][az sdk evtgrd dotnet] in this post. But there are other SDKs in different languages of your choice:
>
> * [JavaScript][az sdk evtgrd js]
> * [Python][az sdk evtgrd python]
> * [Java][az sdk evtgrd java]


## Installing Azure EventGrid SDK Preview ##

At the time of this writing, the new SDK has the version of [`4.0.0-beta.4`][nuget evtgrd new]&ndash;still in preview. With this SDK, you can send and receive event data in the format of CloudEvents. Install the preview version of the SDK by entering the following command:

https://gist.github.com/justinyoo/8282de7244bccca562cd508e64d89470?file=01-dotnet-add-package.sh

Let's deal with the EventGrid data.


## Sending Event Data in CloudEvents Format ##

To use EventGrid related commands in [Azure CLI][az cli], you should install the relevant [extension][az cli extensions] first:

https://gist.github.com/justinyoo/8282de7244bccca562cd508e64d89470?file=02-az-extension-add.sh

Then, get the endpoint of your EventGrid Custom Topic:

https://gist.github.com/justinyoo/8282de7244bccca562cd508e64d89470?file=03-get-endpoint.sh

And, get the access key:

https://gist.github.com/justinyoo/8282de7244bccca562cd508e64d89470?file=04-get-access-key.sh

With both endpoint and access key you got from the commands above, create the EventGrid publisher instance in your Azure Functions code:

https://gist.github.com/justinyoo/8282de7244bccca562cd508e64d89470?file=05-create-publisher.cs

You need more metadata details to send event data in the CloudEvents format.

* `source`: Event Publisher&ndash;It's a URL format in general.
* `type`: Event Type&ndash;It distinguishes events from each other. It usually has a format like `com.example.someevent`.
* `datacontenttype`: Always provide `application/cloudevents+json`.

The SDK takes care of the rest of the metadata that CloudEvents needs.

> If you want to know more about the CloudEvents data spec, visit [this page][ce spec json] and have a read.

With those metadata details, build up the CloudEvents instances and send them to [Azure EvengGrid][az evtgrd].

https://gist.github.com/justinyoo/8282de7244bccca562cd508e64d89470?file=06-publish-event.cs

You can now send the event data in the CloudEvents format to [EventGrid Custom Topic][az evtgrd topic custom].


## Receiving Event Data in CloudEvents Format ##

As mentioned above, [Azure Functions][az fncapp] has the [EventGrid binding extension][az fncapp binding evtgrd] and it doesn't support CloudEvents yet. Therefore, to process the CloudEvents data, you should use [HTTP Trigger][az fncapp trigger http]. And the trigger MUST handle two different types of requests at the same time.

* Responding Event Handler Validation Requests
* Handling Event Data


### Responding Event Handler Validation Requests ###

According to the [CloudEvents Webhook Spec][ce spec webhook], the validation requests uses the `OPTIONS` method/verb and include a request header of `WebHook-Request-Origin` (line #8). Therefore, to respond to the validation request, the header value MUST be sent back to the EventGrid Topic in the response header of `WebHook-Allowed-Origin` (line #9).

https://gist.github.com/justinyoo/8282de7244bccca562cd508e64d89470?file=07-validate-request.cs&highlights=8,9


### Handling Event Data ###

Once the validation request is successful, the Azure Functions endpoint will receive the event data with the `POST` method/verb from there. If you want to use the CloudEvents itself, use the `@event` instance in the following code snippet (line #18). If you only need the `data` part, deserialise it (line #19).

https://gist.github.com/justinyoo/8282de7244bccca562cd508e64d89470?file=08-handle-event.cs&highlights=18,19

You can now receive and process the event data in the CloudEvents format within the Azure Functions code.

---

So far, we have walked through how [Azure Functions][az fncapp] app sends and receive event data formatted in [CloudEvents][ce] for [Azure EventGrid][az evtgrd]. Until the new version of the [binding extension][az fncapp binding evtgrd] is released, we should stick on the approach like this. Hope the new one is released sooner rather than later.


[az cli]: https://docs.microsoft.com/cli/azure/what-is-azure-cli?WT.mc_id=dotnet-12565-juyoo
[az cli extensions]: https://docs.microsoft.com/cli/azure/azure-cli-extensions-list?WT.mc_id=dotnet-12565-juyoo

[az fncapp]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-12565-juyoo
[az fncapp binding evtgrd]: https://docs.microsoft.com/azure/azure-functions/functions-bindings-event-grid?WT.mc_id=dotnet-12565-juyoo
[az fncapp trigger http]: https://docs.microsoft.com/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=csharp&WT.mc_id=dotnet-12565-juyoo

[az evtgrd]: https://docs.microsoft.com/azure/event-grid/overview?WT.mc_id=dotnet-12565-juyoo
[az evtgrd topic custom]: https://docs.microsoft.com/azure/event-grid/custom-topics?WT.mc_id=dotnet-12565-juyoo

[nuget evtgrd legacy]: https://www.nuget.org/packages/Microsoft.Azure.EventGrid/
[nuget evtgrd new]: https://www.nuget.org/packages/Azure.Messaging.EventGrid/

[az sdk evtgrd dotnet]: https://github.com/Azure/azure-sdk-for-net/tree/master/sdk/eventgrid/Azure.Messaging.EventGrid
[az sdk evtgrd js]: https://github.com/Azure/azure-sdk-for-js/tree/master/sdk/eventgrid/eventgrid
[az sdk evtgrd python]: https://github.com/Azure/azure-sdk-for-python/tree/master/sdk/eventgrid/azure-eventgrid
[az sdk evtgrd java]: https://github.com/Azure/azure-sdk-for-java/tree/master/sdk/eventgrid/azure-messaging-eventgrid

[ce]: https://cloudevents.io/
[ce spec json]: https://github.com/cloudevents/spec/blob/v1.0/json-format.md#23-examples
[ce spec webhook]: https://github.com/cloudevents/spec/blob/v1.0/http-webhook.md#4-abuse-protection
