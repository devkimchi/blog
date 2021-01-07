---
title: "Provisioning EventGrid Subscription and LogicApp Handler Using Azure CLI"
slug: provisioning-eventgrid-subscription-and-logicapp-handler-using-azure-cli
description: "This post discusses how to use Azure CLI to provision Azure Event Grid Subscription resource with Logic App as an event handler, which is not supported by ARM template out-of-the-box."
date: "2021-01-06"
author: Justin-Yoo
tags:
- azure-cli
- azure-logic-apps
- azure-eventgrid
- github-actions
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/01/provisioning-eventgrid-subscription-and-logicapp-handler-using-azure-cli-00.png
fullscreen: true
---

You are about to build an event-driven architecture, primarily to integrate existing enterprise applications. In that case, an event broker like [Azure EventGrid][az evtgrd] takes a vital role. The following diagram introduces how Azure EventGrid plays on the Azure Platform.

![Azure Event Grid][image-01]

But As you can see, if you are running hybrid or heterogeneous cloud, either "Custom Events" or "CloudEvents" is your choice.


## Three Pillars of Event-Driven Architecture ##

There are three major players of the event-driven architecture. I've taken the screenshots below from my talk about [CloudEvents][oid ce] at [Open Infra Days Korea][oid] back in 2018.


### Event Publisher ###

It's the source of the event. Any device can be the event publisher as long as it can raise events—the picture below depicted Raspberry Pi as the event publisher.

![Event Publisher][image-02]


### Event Subscriber / Event Handler ###

Strictly speaking, the event subscriber should be distinguished from the event handler. But in general, the event subscriber takes care of the events including processing them, so both terms are often interchangeable. The screenshot below describes how the event subscriber handles the events and processes them for visualising.

![Event Subscriber][image-03]


### Event Broker ###

Because of the events' nature, both event publisher and subscriber can't be directly integrated. Therefore, the event broker should be placed in between. [Azure EventGrid][az evtgrd] is the very player as the event broker. Here's the screenshot showing how [Azure EventGrid][az evtgrd] works between the publisher and subscriber.

![Event Broker][image-04]

> If you're OK with Korean language, you can check my talk [video][oid yt] and [slides][oid ss] about [CloudEvents][ce]. There are no English subtitles provided, unfortunately.


## Provisioning Azure EventGrid Subscription in ARM Template ##

It's OK to provision the Azure EventGrid [Custom Topic][az evtgrd cus topic] through [ARM Template][az evtgrd arm topic]. However, provisioning the Azure EventGrid Subscription using [ARM Template][az evtgrd arm sub] only corresponds with the [System Topic][az evtgrd sys topic], not the Custom Topic you just created without scoping it. Therefore, instead of the ARM template, you should consider [Azure CLI][az cli] to create the subscription.

---

***Updates:***

In fact, you can create the EventGrid Subscription resource and specify the topic using the [ARM Template][az evtgrd arm sub] in two different ways.

### Providing Scope (Recommended) ###

To use the [ARM Template][az evtgrd arm sub] mentioned above, the `scope` attribute is the key. (line #7). Here's the ARM Template written in [Bicep][az bicep]:

https://gist.github.com/justinyoo/8865213b31baeda9f7c1ad258351a039?file=06-az-eventgrid-event-subscription-create-1.bicep&highlights=7


### Providing Nested Resource Type (Doable but NOT Recommended) ###

Alternatively, you can provide the nested resource type like below. In this case, you don't need the `scope` attribute, but as you can see the resource type and name looks more verbose (line #6-7).

https://gist.github.com/justinyoo/8865213b31baeda9f7c1ad258351a039?file=07-az-eventgrid-event-subscription-create-2.bicep&highlights=6-7

Therefore, you can provision Azure EventGrid Subscription resource like above, but let's focus on Azure CLI in this post.

---


## Azure CLI Extensions ##

To provision Azure EventGrid Subscription using [Azure CLI][az cli], a couple of [extensions][az cli extensions] are required to install beforehand. The [Logic][az cli extensions logic] extension is to handle [Azure Logic Apps][az logapp], which will be used as the event handler.

* [EventGrid][az cli extensions eventgrid]
* [Logic][az cli extensions logic]

> Both extensions are currently in preview and will be changed at any time without notice.


## Azure CLI Commands ##

As I would use [Azure Logic App][az logapp] as the event handler, I need to get its endpoint URL. To call the Logic App endpoint, we must know a SAS token for it. As the first step, we should get the resource ID of the Logic App by running the following command, `az logic workflow show`:

https://gist.github.com/justinyoo/8865213b31baeda9f7c1ad258351a039?file=01-az-logic-workflow-show.sh

With this resource Id, use the `az rest` command to get the endpoint containing the SAS token.

https://gist.github.com/justinyoo/8865213b31baeda9f7c1ad258351a039?file=02-az-rest.sh

Now, we got the Logic App endpoint to be added as the event handler. The next step would be to get the resource ID for the EventGrid Topic, by running the following command, `az eventgrid topic show`:

https://gist.github.com/justinyoo/8865213b31baeda9f7c1ad258351a039?file=03-az-eventgrid-topic-show.sh

We got all information we need. Run the `az eventgrid event-subscription create` command to provision the EventGrid Subscription. Let's use the event schema type of [CloudEvents][ce], which is an incubating project of [CNCF][cncf] (line #4).

https://gist.github.com/justinyoo/8865213b31baeda9f7c1ad258351a039?file=04-az-eventgrid-event-subscription-create.sh&highlights=4

So, we finally got the EventGrid Subscription instance towards the EventGrid Custom Topic! And here's the one-liner of all commands above:

https://gist.github.com/justinyoo/8865213b31baeda9f7c1ad258351a039?file=05-one-liner.sh

---

So far, we have walked through how to provision the EventGrid Subscription to EventGrid Custom Topic, using [Azure CLI][az cli]. As this is command-line friendly, you can easily integrate it with any CI/CD pipeline like [GitHub Actions][gh actions].


[image-01]: https://docs.microsoft.com/azure/event-grid/media/overview/functional-model.png?WT.mc_id=devops-12244-juyoo
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/01/provisioning-eventgrid-subscription-and-logicapp-handler-using-azure-cli-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/01/provisioning-eventgrid-subscription-and-logicapp-handler-using-azure-cli-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/01/provisioning-eventgrid-subscription-and-logicapp-handler-using-azure-cli-04.png

[az cli]: https://docs.microsoft.com/cli/azure/what-is-azure-cli?WT.mc_id=devops-12244-juyoo
[az cli extensions]: https://docs.microsoft.com/cli/azure/azure-cli-extensions-list?WT.mc_id=devops-12244-juyoo
[az cli extensions eventgrid]: https://github.com/Azure/azure-cli-extensions/tree/master/src/eventgrid
[az cli extensions logic]: https://github.com/Azure/azure-cli-extensions/tree/master/src/logic

[az bicep]: https://github.com/Azure/bicep

[az logapp]: https://docs.microsoft.com/azure/logic-apps/logic-apps-overview?WT.mc_id=devops-12244-juyoo

[az evtgrd]: https://docs.microsoft.com/azure/event-grid/overview?WT.mc_id=devops-12244-juyoo
[az evtgrd arm topic]: https://docs.microsoft.com/azure/templates/microsoft.eventgrid/topics?WT.mc_id=devops-12244-juyoo
[az evtgrd arm sub]: https://docs.microsoft.com/azure/templates/microsoft.eventgrid/eventsubscriptions?WT.mc_id=devops-12244-juyoo
[az evtgrd sys topic]: https://docs.microsoft.com/azure/event-grid/system-topics?WT.mc_id=devops-12244-juyoo
[az evtgrd cus topic]: https://docs.microsoft.com/azure/event-grid/custom-topics?WT.mc_id=devops-12244-juyoo

[oid]: https://event.openinfradays.kr/2018/about/
[oid ce]: https://event.openinfradays.kr/2018/session1/track_4_0
[oid yt]: https://youtu.be/h2_ZNTXwlVc
[oid ss]: https://www.slideshare.net/openstack_kr/openinfra-days-korea-2018-track-4-cloudevents

[cncf]: https://www.cncf.io/
[ce]: https://cloudevents.io/

[gh actions]: https://docs.github.com/en/free-pro-team@latest/actions
