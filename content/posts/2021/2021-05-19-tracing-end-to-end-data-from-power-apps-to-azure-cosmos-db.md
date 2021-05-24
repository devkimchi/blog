---
title: "Tracing End-to-End Data from Power Apps to Azure Cosmos DB"
slug: tracing-end-to-end-data-from-power-apps-to-azure-cosmos-db
description: "This post discusses how to make use of Application Insights within Azure Functions to trace data end-to-end, and relate them to the concept of OpenTelemetry."
date: "2021-05-19"
author: Justin-Yoo
tags:
- serverless
- observability
- traceability
- open-telemetry
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/05/tracing-end-to-end-data-from-power-apps-to-azure-cosmos-db-00.png
fullscreen: true
---

When you see your cloud-based application architecture, no matter it is microservices architecture or not, many systems are inter-connected and send/receive messages in real-time, near real-time or asynchronously. You all know, in this environment, at some stage, some messages are often failed to deliver to their respective destinations or halted while processing them.

In the cloud environment, components in a system run with their rhythm. Therefore, you should assume that a particular component gets hiccup at any point in time and design the architecture based on this assumption. Therefore, to minimise message loss, you should be able to trace them from one end to the other end. We use the term "Observability" and "Traceability" for it.

In my [previous post][post 1], a citizen dev in a fusion team uses an [Azure Functions][az fncapp] app that enables the OpenAPI capability, and build a [Power Apps][pa] app. This time, I'm going to add a capability that traces the workout data from the Power Apps app to [Azure Cosmos DB][az cosdba] through [Azure Monitor][az monitor] and [Application Insights][az appins]. I will also discuss how this ability is related to the concepts from [Open Telemetry][cncf opentelemetry].

* [Developing Power Apps in Fusion Teams][post 1]
* ***Tracing End-to-End Data from Power Apps to Azure Cosmos DB***
* Putting DevOps and Power Apps Together

> You can find the sample code used in this post at this [GitHub repository][gh sample].


## Scenario ##

Mallee Bulls Fitness Centre provides their members with a [Power Apps][pa] app to record their workout details. Ji Min, the trainer team lead, recently got some feedback from her members that the app gets crashed while putting their workout logs. As she represents the trainers in the fusion team, she started discussing this issue with Su Bin, the pro dev in the team. As a result, Su Bin decided to add a tracing logic into the Function app. Here's the high-level diagram that describes the data processing flow.

![GymLog Telemetry Architecture][image-01]

Let's analyse the diagram based on the [Open Telemetry][cncf opentelemetry] spec.

* The entire data flow from [Power Apps][pa] to [Cosmos DB][az cosdba] is called "[Trace][cncf opentelemetry trace]".
* The whole flow is divided into two distinctive parts by [Azure Service Bus][az svcbus] because both sides are totally different and independent applications (**Publisher** and **Subscriber**). So, these two individual parts are called "[Span][cncf opentelemetry span]". In other words, the Span is a unit of work that handles messages.
  * The Publisher stores the data sent from Power Apps to [Azure Table Storage][az st table], aggregates them through the `publish` action, and send them to [Azure Service Bus][az svcbus].

  > In the picture above, the Publisher consists of three actions, `routine`, `exercise` and `publish`. Although you can split it into three sub Spans, let's use one Span for now.

  * The Subscriber receives the message from [Azure Service Bus][az svcbus], transforms the message and stores it to [Cosmos DB][az cosdba].
* When a message traverses over spans, you need a carrier for metadata so that you can trace the message within the whole Trace. The metadata is called "[Span Context][cncf opentelemetry spancontext]".


## Power Apps Update ##

As mentioned earlier, trace starts from the Power Apps app. Therefore, the app needs an update for the tracing capability. Generate both `correlationId` and `spanId` when tapping the **Start** button and send both to API through the `routine` action.

![Power Apps Canvas - Correlation ID and Span ID][image-02]

By doing so, you know the tracing starts from the Power Apps app side while monitoring, and the first Span starts from it as well. Both `correlationId` and `spanId` travels until the `publish` action is completed. Moreover, the `correlationId` is transferred to the other Span through the Span Context.


## Backend Update ##

As long as the Azure Functions app knows the instrumentation key from an [Application Insights][az appins] instance, it traces almost everything. [OpenTelemetry.NET][cncf opentelemetry dotnet] is one of the [Open Telemetry][cncf opentelemetry] implementations, has recently released [v1.0][cncf opentelemetry dotnet ga] for tracing. Both metrics and logging are close to GA. However, [it doesn't work well with Azure Functions][cncf opentelemetry dotnet issue]. Therefore, in this post, let's manually implement the tracing at the log level, which is sent to [Application Insights][az appins].


### Publisher &ndash; HTTP Trigger ###

When do we take the log?

In this example, the backend APIs consist of `routine`, `exercise` and `publish` actions. Each action stores data to [Azure Table Storage][az st table], by following the event sourcing approach. So, it's good to take logs around the data handling as checkpoints. In addition to that, while invoking the `publish` action, it aggregates the data stored from the previous actions and sends the one to [Azure Service Bus][az svcbus], which is another good point that takes the log as a checkpoint.

All the logging features used in Azure Functions implement the `ILogger` interface. Through this interface, you can store custom telemetry values to Application Insights. Then, what could be the values for the custom telemetry?

* **Event Type**: Action and its invocation result &ndash; `RoutineReceived`, `ExerciseCreated` or `MessageNotPublished`
* **Event Status**: Success or failure of the event &ndash; `Succeeded` or `Failed`
* **Event ID**: Azure Functions invocation ID &ndash; whenever a new request comes in a new GUID is assigned.
* **Span Type**: Type of Span &ndash; `Publisher` or `Subscriber`
* **Span Status**: Current Span status &ndash; `PublisherInitiated`, `SubscriberInProgress` or `PublisherCompleted`
* **Span ID**: GUID assigned to Span each time it is invoked
* **Interface Type**: Type of user interface &ndash; `Test Harness` or `Power Apps App`
* **Correlation ID**: Unique ID for the whole Trace

It could be the bare minimum stored to [Application Insights][az appins]. Once you capture them, you will be able to monitor in which trace (correlation ID) the data flow through which user interface (interface type), span (span type), and event (event type) successfully or not (event status).

Here's the [extension method][gh sample logger] for the `ILogger` interface. Let's have a look at the sample code below that checks in the request data from Power Apps is successfully captured on the `routine` action. Both `correlationId` and `spanId` are sent from Power Apps (line #9-10). The `invocationId` fro the Azure Functions context has become the `eventId` (line #12). Finally, event type, event status, span type, span status, interface type and correlation ID are logged (line #14-17).

https://gist.github.com/justinyoo/9679433b6d886897cc09d8bcf1c8b6de?file=01-create-routine-01.cs&highlights=9-10,12,14-17

The code below shows another checkpoint. Store the request data to Azure Table Storage (line #14). If it's successful, log it (line #18-23). If not, throw an exception, handle it and log the exception details (line #29-34).

https://gist.github.com/justinyoo/9679433b6d886897cc09d8bcf1c8b6de?file=02-create-routine-02.cs&highlights=14,18-23,29-34

In similar ways, the other `exercise` and `publish` actions capture the checkpoint logs.


### Publisher &ndash; Span Context ###

The `publish` action in the Publisher Span doesn't only capture the checkpoint log, but it should also implement Span Context. Span Context contains metadata for tracing, like correlation ID. Depending on the message transfer method, use either the HTTP request header or message envelope. As this system uses Azure Service Bus, use the `ApplicationProperties` dictionary in its message envelope.

Let's have a look at the code for the `publish` action. This part describes that the message body is about the workout details (line #23-24). Other data is stored to `CorrelationId` and `MessageId` properties of the message object (line #26-27) and the `ApplicationProperties` dictionary so that the subscriber application makes use of them (line #30-33). Finally, after sending the message to Azure Service Bus, capture another checkpoint that message has been successfully sent (line #37-42).

https://gist.github.com/justinyoo/9679433b6d886897cc09d8bcf1c8b6de?file=03-publish-routine.cs&highlights=23-24,26-27,30-33,37-42


### Subscriber &ndash; Service Bus Trigger ###

As the tracing metadata is transferred from Publisher via Span Context, Subscriber simply uses it. The following code describes how to interpret the message envelop. Restore the correlation ID (line #10) and Message ID (line #13). And capture another checkpoint whether the message restore is successful or not (line #16-19).

https://gist.github.com/justinyoo/9679433b6d886897cc09d8bcf1c8b6de?file=04-ingest-01.cs&highlights=10,13,16-19

Then, store the message to Azure Cosmos DB (line #12), log another checkpoint (line #16-21). If there's an error while processing the message, handle the exception and capture the checkpoint as well (line #25-30).

https://gist.github.com/justinyoo/9679433b6d886897cc09d8bcf1c8b6de?file=05-ingest-02.cs&highlights=12,16-21,25-30

So far, all paths the data sways have been marked as checkpoints and store the check-in log to Application Insights. Now, how can we check all the traces on [Azure Monitor][az monitor]?


## KUSTO Query on Azure Monitor ##

This time, Ji Min received another feedback that a new error has occurred while storing the workout details with screenshots.

![Power Apps Workout Screen][image-03]
![Power Apps Error Screen][image-04]

As soon as Ji Min shared the pictures with Su Bin, Su Bin wrote a [Kusto query][az monitor kusto] and ran it on Application Insights. Assign the `correlationId` value for tracing (line #1). Then use the custom telemetry values for the query. As all the custom properties start with `customDimensions.prop__`, include them in the `where` clause with the correlation ID for filtering (line #4), and in the `project` clause to select fields that I want to see (line #5-18).

https://gist.github.com/justinyoo/9679433b6d886897cc09d8bcf1c8b6de?file=06-kusto-query.kql&highlights=1,4,5-18

And here's the query result. It says it was OK to receive the exercise data, but it failed to store it to Azure Table Storage.

![Application Insights Kusto Query Result - Failed][image-05]

Now, Su Bin found out where the error has occurred. She fixed the code and deployed the API again, and all is good! The following screenshot shows one of the successful end-to-end tracking logs. A Message sent from Publisher has processed well on the Subscriber side, and the message has become a record based on the logic implemented on the Subscriber side.

![Application Insights Kusto Query Result - Succeeded][image-06]

So, we confirm that the data tracing logic has been implemented by following the [Open Telemetry][cncf opentelemetry] concepts through [Application Insights][az appins]. Ji Min and her trainer crews, and all the members in the gym are now able to know the reference ID for tracing.

---

So far, we've walked through the implementation of data tracing logic with the concept of [Open Telemetry][cncf opentelemetry], from [Power Apps][pa] to [Cosmos DB][az cosdba] through [Application Insights][az appins].

Unfortunately, the [OpenTelemetry.NET][cncf opentelemetry dotnet] doesn't work in Azure Functions as expected for now. But we can still implement the concept through Application Insights for the time being. In the next post, let's try the DevOps journey with Power Apps.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/05/tracing-end-to-end-data-from-power-apps-to-azure-cosmos-db-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/05/tracing-end-to-end-data-from-power-apps-to-azure-cosmos-db-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/05/tracing-end-to-end-data-from-power-apps-to-azure-cosmos-db-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/05/tracing-end-to-end-data-from-power-apps-to-azure-cosmos-db-04.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/05/tracing-end-to-end-data-from-power-apps-to-azure-cosmos-db-05.png
[image-06]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/05/tracing-end-to-end-data-from-power-apps-to-azure-cosmos-db-06.png


[post 1]: /2021/05/12/power-apps-in-fusion-teams/
[post 2]: /2021/05/19/tracing-end-to-end-data-from-power-apps-to-azure-cosmos-db/

[gh sample]: https://github.com/aliencube/GymLog
[gh sample logger]: https://github.com/aliencube/GymLog/blob/main/src/GymLog.FunctionApp/Extensions/LoggerExtensions.cs

[cncf]: https://cncf.io/
[cncf opentelemetry]: https://opentelemetry.io/
[cncf opentelemetry dotnet]: https://opentelemetry.io/docs/net/
[cncf opentelemetry dotnet ga]: https://devblogs.microsoft.com/dotnet/opentelemetry-net-reaches-v1-0/?WT.mc_id=dotnet-28936-juyoo
[cncf opentelemetry dotnet issue]: https://github.com/open-telemetry/opentelemetry-dotnet/issues/1602
[cncf opentelemetry trace]: https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/overview.md#traces
[cncf opentelemetry span]: https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/overview.md#spans
[cncf opentelemetry spancontext]: https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/overview.md#spancontext

[az fncapp]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-28936-juyoo

[az svcbus]: https://docs.microsoft.com/azure/service-bus-messaging/service-bus-messaging-overview?WT.mc_id=dotnet-28936-juyoo
[az cosdba]: https://docs.microsoft.com/azure/cosmos-db/introduction?WT.mc_id=dotnet-28936-juyoo

[az st table]: https://docs.microsoft.com/azure/storage/tables/table-storage-overview?WT.mc_id=dotnet-28936-juyoo

[az appins]: https://docs.microsoft.com/azure/azure-monitor/app/app-insights-overview?WT.mc_id=dotnet-28936-juyoo
[az monitor]: https://docs.microsoft.com/azure/azure-monitor/overview?WT.mc_id=dotnet-28936-juyoo
[az monitor kusto]: https://docs.microsoft.com/azure/azure-monitor/logs/log-query-overview?WT.mc_id=dotnet-28936-juyoo

[pa]: https://powerapps.microsoft.com/?WT.mc_id=dotnet-28936-juyoo
