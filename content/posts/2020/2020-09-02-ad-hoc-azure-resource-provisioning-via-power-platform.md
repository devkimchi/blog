---
title: "Ad-hoc Azure Resource Provisioning via Power Platform"
slug: ad-hoc-azure-resource-provisioning-via-power-platform
description: "This post shows one of the examples how Power Platform increases productivity by simplifying an ad-hoc workflow to provision resources onto Azure."
date: "2020-09-02"
author: Justin-Yoo
tags:
- power-platform
- azure-resource-manager
- provisioning
- productivity
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-00.png
fullscreen: true
---

Throughout this blog post series, let's find out an example how [Power Platform][pw platform] increases productivity by simplifying an ad-hoc workflow to provision resources like [Azure VM][az vm] onto Azure.

* [Auto-installing Applications on Azure VM with Chocolatey for Live Streaming][post prev]
* ***Ad-hoc Azure Resource Provisioning via Power Platform***

In my [previous post][post prev], we walked through how to install live-streaming related applications to Azure Windows VM while provisioning it. By the way, this type of VM provisioning is required ad-hoc basis, rather than a regular schedule. We create an [ARM template][az arm template] for it, but we never know when it is used. The nature of ad-hoc workflows is like this. We may use it again, but we never know when it will be. We may not be ready to run when it needs to be run.

[Power Apps][pw apps] is the right fit to handle this sort of running ad-hoc workflows on mobile devices. This post shows a glimpse of an idea how [Power Apps][pw apps] and [Power Automate][pw automate] handles Azure resource provisioning so that your IT pros in your organisation can create ad-hoc resources without having to get a DevOps engineer.


## One Parameter Rules Power Automate Workflow? ##

The number of parameters from Power Apps is determined by the Power Apps trigger on Power Automate. If the number of parameters or parameter names used in a Power Automate workflow is too many, changes often, or is non-deterministic, this approach would be useful.

First of all, add a [`Compose` action][pw automate compose] and change its name to `ParametersInJson`. Then create a parameter for it, which will be named to `ParametersInJson_Inputs`.

![][image-01]

Add the function expression, `json(triggerBody()?['ParametersInJson_Inputs'])`, to the `Inputs` field like this:

![][image-02]

If we handle all the parameters passed from Power Apps in this way, we only have one parameter but take all values passed from Power Apps flexibly. This approach also avoids Power Apps from keeping remove and re-connect Power Automate over and over again whenever the parameters are updated.


## Power Automate Workflow &ndash; Azure Resource Provisioning ##

With the flexible parameter passed from Power Apps, run the ARM template deployment using the [Azure Resource Manager deployment action][pw automate arm deploy]. All the parameters used for this action are like:

* `outputs('ParametersInJson')?['resourceGroupName']`,
* `outputs('ParametersInJson')?['deploymentName']`,
* `outputs('ParametersInJson')?['vmName']`,
* `outputs('ParametersInJson')?['vmAdminUsername']` and
* `outputs('ParametersInJson')?['vmAdminPassword']`.

Also, notice that the `Wait for Deployment` field value is set to `No`. I'll discuss it soon.

![][image-03]

Once this action is run, the result should be returned to Power Apps through the [Response action][pw automate response]. The following screenshot shows how it uses the output of the ARM deployment action with JSON schema.

![][image-04]

Now, we've created the workflow for Azure resource provisioning.

By the way, we need to consider the nature of this action. It takes from 30 seconds to 40 minutes or longer that completes the resource provisioning. As Power Apps can't wait for it, the workflow should be running asynchronously. Did you remember that the `Wait for Deployment` field has been set to `No` in the previous action? The actual response has the status code of `201`, not `200`, because of this option.

How can we check the result of the resource provisioning? Let's build another workflow for it.


## Power Automate Workflow &ndash; Azure Resource Provisioning Status Check ##

This time, let's build another workflow that checks the resource provisioning status. It only checks the status. We also use the same approach above to take the parameters from the Power App instance.

And let's use the [action to check the provisioning status][pw automate arm status]. All the relevant variables look like:

* `outputs('ParametersInJson')?['resourceGroupName']` and
* `outputs('ParametersInJson')?['deploymentName']`

![][image-05]

The last action is the [Response action][pw automate response] that sends the action response back to Power Apps.

![][image-04]

We've now got two workflows for the resource provisioning. Let's build the Power Apps now.


## Power Apps &ndash; Ad-hoc Azure Resource Provisioning ##

The layout of the app that IT pros in your organisation will use might look like the following. It accepts five parameters from the user, which will be used for both Power Automate workflows. One button starts the resource provisioning, and the other button checks the provisioning status.

![][image-06]

Connect the resource provisioning workflow to the `Provision!` button and define the button action like below.

![][image-07]

Note that we use the [`Set()` function][pw apps set] this time. With this function, we create a temporary variable of `request` and assign the JSON object as its value. Then, the `request` value is sent to the Power Automate workflow via the `CreateResource()` function. The `request` object will be decomposed to a JSON object in the Power Automate workflow. And the response of this function is stored to the `result` collection, using the [`ClearCollect()` function][pw apps clearcollect].

https://gist.github.com/justinyoo/90c6e7a93fc0f957aa35751a8fee32f4?file=01-create-resource.txt

As mentioned above, the resource is not instantly provisioned as soon as we tap the `Provision!` button. Therefore, we should use the `Status` button to check the provisioning status.

![][image-08]

Similar to the approach above, we use the `Set()` function to initialise the `status` variable and send it to Power Automate through the `CheckProvisioningStatus()` function. Then the result will be stored to the `result` collection.

https://gist.github.com/justinyoo/90c6e7a93fc0f957aa35751a8fee32f4?file=02-check-provisioning-status.txt

> I use the `Status` button for simplicity. But you can make use of the [`Timer` control][pw apps timer] for a more meaningful way, which is beyond the discussion of this article.

Finally set the label control to take the result from the workflow like `First(result).properties.provisioningState`, using the [`First()` function][pw apps first].

![][image-09]

We've now got the Power App, too! Let's run the Power App with the actual value.

![][image-10]

The first response from the provisioning will be like this:

![][image-11]

In the middle of the provisioning, the status will look like this:

![][image-12]

And after the resource provisioning is complete, the status will look like this:

![][image-13]

---

So far, we've walked through how we built Power Automate workflows and Power Apps for ad-hoc Azure resource provisioning. We only used one use case here for simplicity, but there are more complex real-world examples with many ad-hoc scenarios in your business domain. If you can materialise those scenarios based on priority and frequency, it will increase productivity for sure.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-04.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-05.png
[image-06]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-06.png
[image-07]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-07.png
[image-08]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-08.png
[image-09]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-09.png
[image-10]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-10.png
[image-11]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-11.png
[image-12]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-12.png
[image-13]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-13.png

[post prev]: /2020/08/26/app-provisioning-on-azure-vm-with-chocolatey-for-live-streaming/

[az vm]: https://docs.microsoft.com/azure/virtual-machines/windows/overview?WT.mc_id=devkimchicom-blog-juyoo

[az arm template]: https://docs.microsoft.com/azure/azure-resource-manager/templates/overview?WT.mc_id=devkimchicom-blog-juyoo

[pw platform]: https://powerplatform.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo

[pw apps]: https://powerapps.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo
[pw apps set]: https://docs.microsoft.com/powerapps/maker/canvas-apps/functions/function-set?WT.mc_id=devkimchicom-blog-juyoo
[pw apps clearcollect]: https://docs.microsoft.com/powerapps/maker/canvas-apps/functions/function-clear-collect-clearcollect?WT.mc_id=devkimchicom-blog-juyoo#clearcollect
[pw apps first]: https://docs.microsoft.com/powerapps/maker/canvas-apps/functions/function-first-last?WT.mc_id=devkimchicom-blog-juyoo
[pw apps timer]: https://docs.microsoft.com/powerapps/maker/canvas-apps/controls/control-timer?WT.mc_id=devkimchicom-blog-juyoo

[pw automate]: https://flow.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo
[pw automate compose]: https://docs.microsoft.com/power-automate/data-operations?WT.mc_id=devkimchicom-blog-juyoo#use-the-compose-action
[pw automate arm deploy]: https://docs.microsoft.com/connectors/arm/?WT.mc_id=devkimchicom-blog-juyoo#create-or-update-a-template-deployment
[pw automate arm status]: https://docs.microsoft.com/connectors/arm/?WT.mc_id=devkimchicom-blog-juyoo#read-a-template-deployment
[pw automate response]: https://docs.microsoft.com/azure/connectors/connectors-native-reqres?WT.mc_id=devkimchicom-blog-juyoo#add-a-response-action
