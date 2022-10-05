---
title: "Creating Custom Connector from Azure Functions with Swagger"
slug: creating-custom-connector-from-azure-functions-with-swagger
description: "This post shows how to create a custom connector from Swagger document, automatically generated from Azure Functions instance on-the-fly, and how to apply the custom connector to Power Automate and Power Apps."
date: "2020-07-15"
author: Justin-Yoo
tags:
- azure-functions
- swagger
- custom-connector
- power-platform
cover: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-00.png
fullscreen: true
---

One of the benefits using the [Open API Extension for Azure Functions][post prev] is that it significantly increases the discoverability of the Azure Function app. By taking this benefits, we can easily create a [custom connector][az cuscon] that is integrated with either [Azure Logic Apps][az logapp] or [Power Platform][power platform]. Throughout this post, I'm going to discuss how to integrate the [Open API extension][gh openapi] with [Azure Functions][az func], create a [custom connector][az cuscon], and apply the connector to [Power Automate][power automate] and [Power Apps][power apps].


## Sample Azure Function App ##

Here's the sample Azure Function app that has two endpoints of `/feeds/items` and `/feeds/item` (line #7, 15).

https://gist.github.com/justinyoo/2b4bc731ff8f2cdb5e80e28bd7dff9e7?file=01-feed-reader.cs&highlights=7,15

When you run the app, it shows the two endpoints like below, obviously:

![][image-01]


## Installing NuGet Package ##

First of all, install the NuGet package to enable Open API document feature onto the Azure Functions app.

https://gist.github.com/justinyoo/2b4bc731ff8f2cdb5e80e28bd7dff9e7?file=02-install-nuget-package.sh


## Installing Boilerplate Codes ##

In fact, we don't need to do anything further to write the boilerplate codes any longer. Therefore, simply build the app and run it locally. Then, you'll be able to see the additional three endpoints that represent Open API document and UI.

![][image-02]

Pick up the endpoint of `http://localhost:7071/api/swagger/ui` and run it on your web browser.

![][image-03]

Oops, we've got the Swagger UI page, but no endpoint is showing up, because we haven't let the Swagger know.


## Decorating Endpoints for Open API Extension ##

Use the decorators to let the Swagger know they are endpoints. We use the decorators of `OpenApiOperation`, `OpenApiRequestBody` and `OpenApiResponseBody`(line #2-4, 13-15).

https://gist.github.com/justinyoo/2b4bc731ff8f2cdb5e80e28bd7dff9e7?file=08-add-decorators.cs&highlights=2-4,13-15

Once completed, build the app again and run it locally. Now you're able to see the Swagger UI with complete endpoints!

![][image-04]

So, we've got the Open API integrated Azure Function app. Deploy it to [Azure][az] and check out the Swagger UI page on Azure.

![][image-05]

We're good to go for [custom connectors][az cuscon]! Let's move on.


## Creating Custom Connector ##

Once you create a custom connector, it is shared with both [Power Automate][power automate] and [Power Apps][power apps]. Therefore, I'm going to create it on the [Power Automate][power automate] site. First of all, let's use the Open API document URL like below, to create the one:

![][image-06]

If you see the error message below, don't panic. It happens sometimes.

![][image-07]

Then, just save the document from the Azure Function app site, and upload it.

![][image-08]

Once you completed uploading, then everything goes smoothly, unless your Swagger document has some weird stuff. However, in most cases, as the extension was designed for the custom connector, it should just work. Click the `✅ Create Connector` button to complete.

![][image-09]

Let's have a test whether the connector works OK or now. Create an API connection on the `4. Test` tab.

> Make sure the concept that a `Connector` is an interface, and the `Connection` to the `Connector` is an actual implementation for your app to use.

![][image-10]

While you're creating the connection, you're asked to put the Azure Functions API key. Enter the key and complete the connection process.

![][image-11]

Now, you've got the connection in place. Let's move on for the test. The input fields exactly follow from the Swagger definition. Enter the value to each field and click the `Test Operation` button.

![][image-12]

We've got the test passed!

![][image-13]

Let's move on to create a Power Automate flow.


## Creating Power Automate Flow with Custom Connector ##

Follow the step, as Power Apps will use the flow we're going to create. Choose `Instant Flow`.

![][image-14]

Then choose Power App as a trigger.

![][image-15]

Now, we have the visual designer for weaving the flow. Click the `➕ New Step` button.

![][image-16]

Search up `ATOM`, and you'll be able to see the custom connector that you just created with two endpoints. Select the action that picks up one sing feed item.

![][image-17]

When the action appears, it asks to put the data into the field, which we saw while testing the connector above. Enter the value to each field.

![][image-18]

The purpose of this flow is to post a feed item from YouTube to social media. Let's choose the Twitter action.

![][image-19]

In the body field, enter the data like below, with values populated from the custom connector.

![][image-20]

The last action is `Response` that returns the result to Power Apps.

![][image-21]

Set the response body with the whole body from the custom connector.

![][image-22]

We're getting there! Save the flow and test it. Click the `Test` button on the upper right corner, choose the option and click the `Save & Test` button.

![][image-23]

This screen confirms which connection is used. Click the `Continue` button to proceed.

![][image-24]

If everything is OK, we'll be able to see the screen below, which indicates success.

![][image-25]

Let's have a look at the details of the workflow. Every action has run successfully. Copy the response object to use later.

![][image-26]

When you check it on Twitter, it's posted well. Don't get bothered with the Twitter handle. It's a fictitious company called fairdin.com.

![][image-27]

As the final step on the flow, we need to ensure that Power App can clearly understand the response payload structure. We copied the response payload above. With this payload, create the JSON schema. Click the `Generate from Sample` button.

![][image-28]

Paste the JSON response payload and click `Done`.

![][image-29]

Now, we got the JSON schema for the response payload.

![][image-30]

Save it, and we're done now! Let's move onto Power Apps.


## Connecting Power Automate to Power Apps ##

We're going to connect the Power Automate flow to the Power App we're creating. Have the new app canvas.

![][image-31]

Put one button control, two label controls and one image control onto the canvas.

![][image-32]

We expect that, when you tap the button, it triggers the Power Automate flow. Click the button control, select the `Action` tab at the menu bar at the top. Choose `Power Automate` and select the Power Automate flow that we just created.

![][image-33]

As soon as the Power Automate flow is connected to the Power App, it asks to completes the function. Enter `ClearCollect(result, AmplifyingaRandomYouTubeContent.Run())`. It means that the button runs the Power Automate, represented by the `AmplifyingaRandomYouTubeContent()` function, gets the response and saves it to the `result` collection.

![][image-34]

Now, update the rest controls to display the Power Automate execution result. Apply the formula to each control:

* Upper label control: `First(result).title`
* Lower label control: `First(result).link`
* Image control: `First(result).thumbnailLink`

Then run the Power App, and you'll be able to see the result!

![][image-35]

And Twitter has the same result posted!

![][image-36]

---

So far, we have done:

1. Implementing [Open API extension][gh openapi] on your [Azure Functions][az func] app so that it generates the Swagger document on-the-fly,
2. Building a [custom connector][az cuscon] for [Power Automate][power automate] with this Swagger document
3. Integrating the [custom connector][az cuscon] to [Power Automate][power automate], and
4. Connecting the [Power Automate][power automate] flow to [Power Apps][power apps].

By implementing a simple extension to Azure Functions, it significantly increases its extendability and discoverability. Let's leverage this for your [Power Platform][power platform] app building.


[image-01]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-01.png
[image-02]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-02.png
[image-03]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-03.png
[image-04]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-04.png
[image-05]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-05.png
[image-06]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-06.png
[image-07]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-07.png
[image-08]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-08.png
[image-09]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-09.png
[image-10]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-10.png
[image-11]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-11.png
[image-12]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-12.png
[image-13]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-13.png
[image-14]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-14.png
[image-15]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-15.png
[image-16]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-16.png
[image-17]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-17.png
[image-18]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-18.png
[image-19]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-19.png
[image-20]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-20.png
[image-21]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-21.png
[image-22]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-22.png
[image-23]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-23.png
[image-24]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-24.png
[image-25]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-25.png
[image-26]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-26.png
[image-27]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-27.png
[image-28]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-28.png
[image-29]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-29.png
[image-30]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-30.png
[image-31]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-31.png
[image-32]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-32.png
[image-33]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-33.png
[image-34]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-34.png
[image-35]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-35.png
[image-36]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-36.png


[post prev]: /2019/02/02/introducing-swagger-ui-on-azure-functions/

[gh openapi]: https://github.com/aliencube/AzureFunctions.Extensions
[gh openapi docs openapi]: https://github.com/aliencube/AzureFunctions.Extensions/blob/dev/docs/openapi.md
[gh openapi release]: https://github.com/aliencube/AzureFunctions.Extensions/releases

[az]: https://azure.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo
[az func]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=devkimchicom-blog-juyoo
[az logapp]: https://docs.microsoft.com/azure/logic-apps/logic-apps-overview?WT.mc_id=devkimchicom-blog-juyoo
[az apim]: https://docs.microsoft.com/azure/api-management/api-management-key-concepts?WT.mc_id=devkimchicom-blog-juyoo
[az cuscon]: https://docs.microsoft.com/connectors/custom-connectors/?WT.mc_id=devkimchicom-blog-juyoo

[power platform]: https://powerplatform.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo
[power automate]: https://flow.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo
[power apps]: https://powerapps.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo
