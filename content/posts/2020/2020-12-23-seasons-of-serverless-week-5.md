---
title: "Seasons of Serverless Week 5 Challenge - Tteok-guk for The New year"
slug: seasons-of-serverless-week-5
description: "This post is a part of the #SeasonsOfServerless challenge, for the 5th week. It shows a sample solution to cook tteok-guk, the traditional Korean food for The New Year, using Azure Durable Functions, Logic Apps, Power Automate and Power Apps."
date: "2020-12-23"
author: Justin-Yoo
tags:
- azure-functions
- azure-logic-apps
- power-automate
- power-apps
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2020/12/seasons-of-serverless-week-5-00.png
fullscreen: true
---

> This article is part of [#SeasonsOfServerless][devto sos].
>
> Each week we will publish a challenge co-created by [Azure Advocates][ms ca] with some amazing [Student Ambassadors][ms lsa] around the world. Discover popular festive recipes and learn how Microsoft Azure empowers you to do more with [Azure Serverless services][az serverless]! 🍽 😍.
>
> Explore our serverless [Resources][gh sos resources] and learn how you can contribute solutions [here][gh sos contribute].

* Week 1: [The Perfect Holiday Turkey - North America][devto sos week1]
* Week 2: [Lovely Ladoos - India][devto sos week2]
* Week 3: [The Longest Kebab - Europe / Turkey][devto sos week3]
* Week 4: [A Big Barbeque! - South America / Brazil][devto sos week4]
* Week 5: ***Tteok-guk for The New Year - Asia / Korea***
* Week 6: TBD
* Week 7: TBD

In Korea, when New Year begins, everyone eats tteok-guk (rice cake soup). There are various shapes of tteok, but especially for greeting New Year, garae-tteok is the most popular to make the soup.

As garae-tteok has a long and cylindrical shape, people wish to live long, by eating tteok-guk. When cooking tteok-guk, the garae-tteok is sliced into small pieces, which look like coins. This coin-like shape is believed to bring wealth.

![](https://github.com/justinyoo/Seasons-of-Serverless/blob/main/graphics/2020-12-21/tteokguk.jpg?raw=true)


## Challengers & Chefs ##

Here are the challengers, aka Chefs.

* Justin Yoo: [Microsoft Cloud Advocate][author justin]
* You Jin Kim: [Microsoft Learn Student Ambassador][author youjin]
* Hong Min Kim: [Microsoft Learn Student Ambassador][author hongmin]
* Aaron Roh: [Microsoft Learn Student Ambassador][author aaron]


## Ingredients (for 4 People) ##

Cooking tteok-guk is fairly straightforward. Here is the list of ingredients for four people.

* Garae-tteok: 400g
* Diced beef: 100g
* Water: 10 cups
* Eggs: 2
* Spring onion: 1
* Minced garlic: 1 tablespoon
* Soy sauce: 2 tablespoon
* Sesame oil: 1 tablespoon
* Olive oil: 1 tablespoon
* Salt and pepper


## Recipe ##

The simplest way to cook tteok-guk is the following:

1. Slice garae-tteok into small pieces – no thicker than 5 mm.
   * You can buy sliced garae-tteok.
   * But in this case, put the sliced garae-tteok into a bowl of water for about 30 mins.
2. Slice spring onion.
3. At high heat, stir-fry the diced beef with sesame oil and olive oil until the beef surface goes brown.
4. Put the water into the wok and boil for about 30 mins with medium heat.
5. While boiling, remove bubbles from the water from time to time.
6. Get the eggs beaten.
7. After the 30 mins, put the minced garlic and soy sauce into the boiled soup. Add some salt, if necessary.
8. Add the beaten egg and sliced spring onion.
9. Serve the soup with pepper drizzled on top.

[You Jin][author youjin] depicted the recipe in the flow-chart format (some letters are in Korean).

![Flow chart][image-01]


## Recipe Steps Implementation ##

As each step takes time to complete, the prep time is simulated by a timer. [Azure Durable Function][az func durable] offers the [Timer feature][az func durable timer], which perfectly fits this case.

Each [Ambassador][ms lsa] chose three steps respectively and wrote Function app with their preferred programming language. As a result, [You Jin][author youjin] built [Step 4][gh sos step4], [Step 6][gh sos step6] and [Step 8][gh sos step8] in JavaScript, [Hong Min][author hongmin] wrote [Step 2][gh sos step2], [Step 5][gh sos step5] and [Step 7][gh sos step7] in C#, and [Aaron][author aaron] implemented [Step 1][gh sos step1], [Step 3][gh sos step3] and [Step 9][gh sos step9] in Python.


## Recipe Steps Orchestration ##

While [Durable Function][az func durable] was used for each step, [Azure Logic Apps][az logapp] was used for their overall orchestration. Based on the flow-chart above, the Logic App is triggered by an HTTP request, passes the payload to [Step 1 (Slicing Garae-tteok)][gh sos step1], [Step 2 (Slicing Spring Onion)][gh sos step2], [Step 3 (Stir-fry Beef)][gh sos step3] and [Step 6 (Beating Eggs)][gh sos step6] and process them at the same time. Then, the response from each step is aggregated to move forward. It's a typical pattern called [Fan-Out/Fan-In or Scatter-Gather][fanout fanin].

![Logic App: Fan-out/Fan-in][image-02]

After the aggregation, [Run the Step 4 (Boiling)][gh sos step4]. While boiling, bubbles randomly occur, which should be removed ([Step 5][gh sos step5]).

![Logic App: Boiling & De-bubbling][image-03]

Then, add garlic and sauce ([Step 7][gh sos step7]), Add beaten eggs and spring onion ([Step 8][gh sos step8]), and finally serve the soup by [adding salt and pepper (Step 9)][gh sos step9]. Step 9 returns a random tteok-guk image from [Azure Blob Storage][az st blob], and the image is sent to the designated email.

![Logic App: Rest of Steps][image-04]

At this stage, we have to consider the [two-minute limitation of Logic Apps][az logapp limit] during this execution. Therefore, instead of using the [HTTP action][az logapp http], we used the [HTTP webhook action][az logapp webhook] to overcome this restriction. The following screenshot shows the end-to-end execution result.

![Logic App: Run Result][image-05]

As you can see the last action, the tteok-guk image is sent to the given email like this:

![Email: Run Result][image-06]


## Recipe Automatic Build and Deployment ##

Both [Durable Function][az func durable] apps and [Logic App][az logapp] orchestration are all automatically built and deployed to Azure via [GitHub Actions][gh actions]. You can find more details at the following links. As we used [Bicep][az bicep] to build ARM template, you might like to know more about it. [This link][post bicep] would be helpful, if you like.

* [ARM Templates via Bicep][gh bicep]
* [GitHub Actions workflow][gh workflow]


## Power Automate for Mobile App Integration ##

The first plan was to migrate the Logic App to [Power Automate][pw automate] flow. However, we eventually decided to separate Power Automate side from Logic App side. Within the Power Automate workflow, we call the Logic App workflow using the webhook action. As [Power Apps][pw apps] calls this [Power Automate][pw automate] workflow, it sends the result back to the Power App through the [push notification][pw apps push].

![Power Automate: Workflow][image-07]

In order to use the push notification feature on [Power Automate][pw automate], we should slightly update the Logic App. The final action on the Logic App is not only sending an email but also calling back to the Power Automate with the image URL.

![Logic App: Workflow Update][image-08]


## Power Apps Build ##

To call [Power Automate][pw automate] workflow with parameters, we used [Power Apps][pw apps]. The following screenshot shows which controls we used in the Power Apps canvas.

![Power Apps: Canvas][image-09]

Once completed, publish the [Power Apps][pw apps] and run it on our mobile phone. And here's the result.

https://youtu.be/AJlCv2pAc8g

---

So far, we have built an end-to-end solution to cook tteok-guk, using [Azure serverless][az serverless] services including [Power Platform][pw platform]. The entire source code can be fount at this repository:

* [Solution Repository][gh sample]

As this repository also contains the sample [Power Apps][pw apps] and [Power Automate][pw automate] workflow, download the zip files and import them to your [Power Platform][pw platform] and run them. It will be super easy! If you like to challenge the other recipes, check out this page, [#SeasonsOfServerless][devto sos]!


[image-01]: https://raw.githubusercontent.com/justinyoo/Seasons-of-Serverless/solution/solutions/2020-12-21/flowchart.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/12/seasons-of-serverless-week-5-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/12/seasons-of-serverless-week-5-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/12/seasons-of-serverless-week-5-04.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/12/seasons-of-serverless-week-5-05.png
[image-06]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/12/seasons-of-serverless-week-5-06.jpg
[image-07]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/12/seasons-of-serverless-week-5-07.png
[image-08]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/12/seasons-of-serverless-week-5-08.png
[image-09]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/12/seasons-of-serverless-week-5-09.png

[devto sos]: https://dev.to/azure/azure-advocates-seasons-of-serverless-join-our-virtual-festive-potluck-53m6
[devto sos week1]: https://dev.to/azure/seasonsofserverless-solution-1-developing-the-perfect-holiday-turkey-2p3f
[devto sos week2]: https://dev.to/azure/seasonsofserverless-solution-2-developing-lovely-ladoos-3ggh
[devto sos week3]: https://dev.to/azure/week-3
[devto sos week4]: https://dev.to/azure/week-4
[devto sos week6]: https://dev.to/azure/week-6
[devto sos week7]: https://dev.to/azure/week-7

[post bicep]: /tag/bicep/

[author justin]: https://twitter.com/justinchronicle
[author youjin]: https://github.com/u0jin
[author hongmin]: https://github.com/hongman
[author aaron]: https://www.linkedin.com/in/aaronroh/

[gh sample]: https://github.com/justinyoo/Seasons-of-Serverless
[gh actions]: https://docs.github.com/en/free-pro-team@latest/actions
[gh bicep]: https://github.com/justinyoo/Seasons-of-Serverless/blob/solution/solutions/2020-12-21/Resources/azuredeploy.bicep
[gh workflow]: https://github.com/justinyoo/Seasons-of-Serverless/blob/solution/.github/workflows/main.yaml

[gh sos resources]: https://github.com/microsoft/Seasons-of-Serverless/blob/main/RESOURCES.md
[gh sos contribute]: https://github.com/microsoft/Seasons-of-Serverless/blob/main/CONTRIBUTING.md
[gh sos step1]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-1
[gh sos step2]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-2
[gh sos step3]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-3
[gh sos step4]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-4
[gh sos step5]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-5
[gh sos step6]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-6
[gh sos step7]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-7
[gh sos step8]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-8
[gh sos step9]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-9

[ms ca]: https://developer.microsoft.com/advocates/?WT.mc_id=academic-10291-cxa
[ms lsa]: https://studentambassadors.microsoft.com/?WT.mc_id=academic-10291-cxa

[az bicep]: https://github.com/azure/bicep

[az serverless]: https://azure.microsoft.com/solutions/serverless/?WT.mc_id=academic-10291-cxa
[az st blob]: https://docs.microsoft.com/azure/storage/blobs/storage-blobs-introduction?WT.mc_id=academic-10291-cxa

[az func]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=academic-10291-cxa
[az func durable]: https://docs.microsoft.com/azure/azure-functions/durable/durable-functions-overview?WT.mc_id=academic-10291-cxa
[az func durable timer]: https://docs.microsoft.com/azure/azure-functions/durable/durable-functions-timers?WT.mc_id=academic-10291-cxa

[az logapp]: https://docs.microsoft.com/azure/logic-apps/logic-apps-overview?WT.mc_id=academic-10291-cxa
[az logapp limit]: https://docs.microsoft.com/azure/logic-apps/logic-apps-limits-and-config?WT.mc_id=academic-10291-cxa#http-limits
[az logapp http]: https://docs.microsoft.com/azure/logic-apps/logic-apps-workflow-actions-triggers?WT.mc_id=academic-10291-cxa#http-action
[az logapp webhook]: https://docs.microsoft.com/azure/logic-apps/logic-apps-workflow-actions-triggers?WT.mc_id=academic-10291-cxa#webhooks-and-subscriptions

[pw platform]: https://powerplatform.microsoft.com/?WT.mc_id=academic-10291-cxa
[pw automate]: https://flow.microsoft.com/?WT.mc_id=academic-10291-cxa
[pw apps]: https://powerapps.microsoft.com/?WT.mc_id=academic-10291-cxa
[pw apps push]: https://docs.microsoft.com/connectors/powerappsnotification/?WT.mc_id=academic-10291-cxa

[fanout fanin]: https://www.enterpriseintegrationpatterns.com/patterns/messaging/BroadcastAggregate.html
