---
title: "Auto-installing Applications on Azure VM with Chocolatey for Live Streaming"
slug: app-provisioning-on-azure-vm-with-chocolatey-for-live-streaming
description: "This post shows how to install applications for live streaming, through Chocolatey, while provisioning an Azure Windows Virtual Machine."
date: "2020-08-26"
author: Justin-Yoo
tags:
- azure-vm
- chocolatey
- provisioning
- configuration
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/app-provisioning-on-azure-vm-with-chocolatey-for-live-streaming-00.png
fullscreen: true
---

Throughout this post series, let's find out an example how [Power Platform][pw platform] increases productivity by simplifying an ad-hoc workflow to provision resources like [Azure VM][az vm] onto Azure.

* ***Auto-installing Applications on Azure VM with Chocolatey for Live Streaming***
* [Ad-hoc Azure Resource Provisioning via Power Platform][post next]

Everything has gone. I mean all off-line meetups and conferences disappeared. Instead, they have gone virtual &ndash; online meetups and conferences. For community events, they have two options &ndash; one that purchases a solution for online events, and the other that build a live streaming solution by themselves. If you are a community event organiser and running a live streaming session by yourself, it doesn't really matter whether you install all necessary applications on your computer or not. However, if the event scales out, which includes inviting guests and/or sharing screens, it could be challenging unless your computer has relatively high spec enough.

For this case, there are a few alternatives. One option is to use a virtual machine (VM) on the Cloud. A VM instance can be provisioned whenever necessary, then destroyed whenever no longer required. However, this approach also has a caveat from the "live streaming" point of view. Every time you provision the VM instance, you should install all the necessary applications by hand. If this is not happening very often, it may be OK. But it's still cumbersome to manually install those apps. Throughout this post, I'm going to discuss how to automatically install live streaming related software using [Chocolatey][chocolatey] during the provision of Azure [Windows VM][az vm].

> The sample code used in this post can be found at this [GitHub repository][gh sample].


## Acknowledgement ##

Thanks [Henk Boelman][henk tw] and [Frank Boucher][frank tw]! Their awesome blog posts, [Henk's one][henk blog] and [Frank's one][frank blog] helped a lot to me set this up.


## Installing Live Streaming Applications ##

As we're using a Windows VM, we need those applications for live streaming.

* [Microsoft Edge (Chromium)][ms edge]: As of this writing, Chromium-based Edge is not installed as default. Therefore, it's good to update to this version.
* [OBS Studio][obs]: Open source application for live streaming.
* [OBS Studio NDI Plug-in][obs ndi]: As OBS itself doesn't include the NDI feature, this plug-in is required for live streaming.
* [Skype for Content Creators][skype]: This version of Skype can enable the [NDI][ndi] feature. With this feature enabled, we can capture screens from all participants and shared screens, respectively.

These are the bare minimum for live streaming. Let's compose a PowerShell script that installs them via Chocolatey. First of all, we need to install Chocolatey using the downloadable installation script (line #2). Then, install the software using Chocolatey (line #5-8). The command may look familiar if you've used a CLI-based application package management tool like `apt` or `yum` from Linux, or `brew` from Mac.

https://gist.github.com/justinyoo/acdfc3c854f21f4e10f56e3d9d75e4c7?file=01-install.ps1&highlights=2,5-8

So, if this installation script can be executable while provisioning the Windows VM instance on Azure, we can always use the fresh VM with newly installed applications.


## Provisioning Azure Windows VM ##

Now, let's provision a Windows VM on Azure. Instead of creating the instance on Azure Portal, we can use the [ARM template][az arm] for this. Although there are thousands of ways using the ARM template, let's use the [quick start templates][az quickstart] as a starting point. Based on this template, we can customise the template for our live streaming purpose. We use the template, [Deploy a simple Windows VM][az quickstart vm], and update it. Here is the template. I omitted details for brevity, except VM specs like [VM size][az vm size] (line #43) and [VM image details][az vm image] (line #48-51).

https://gist.github.com/justinyoo/acdfc3c854f21f4e10f56e3d9d75e4c7?file=02-azure-vm.json&highlights=43,48-51

If you want to see the full ARM template, click the following link to GitHub.

[See ARM Template in full][gh sample arm]


### Custom Script Extension ###

We've got the VM instance ready. However, we haven't figured out how to run the PowerShell script during the provision. To run the custom script, add this [extension][az vm custom script] to the ARM template. The custom script in the template looks below. The most important part of this template is the `property` value. Especially, pay attention to both `fileUris` (line #16) and `commandToExecute` (line #19).

https://gist.github.com/justinyoo/acdfc3c854f21f4e10f56e3d9d75e4c7?file=03-arm-vm-extension.json&highlights=16,19

* `fileUris` indicates the location of the custom script. The custom script MUST be publicly accessible like GitHub URL or [Azure Blob Storage][az storage blob] URL.
* `commandToExecute` is the command to execute the custom script. As we use the PowerShell script downloaded externally, add the `-ExecutionPolicy Unrestricted` parameter to loosen the permission temporarily. `./install.ps1` is the filename of the executing script from the URL.


### ARM Template Execution ###

Once everything is done, run the ARM template for deployment. Here's the PowerShell command:

https://gist.github.com/justinyoo/acdfc3c854f21f4e10f56e3d9d75e4c7?file=04-arm-deploy.ps1

And, here's the Azure CLI command:

https://gist.github.com/justinyoo/acdfc3c854f21f4e10f56e3d9d75e4c7?file=05-arm-deploy.sh

If you're lazy enough, click the following button to run the deployment template directly on Azure Portal.

[![Deploy to Azure](http://azuredeploy.net/deploybutton.png)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fdevkimchi%2FLiveStream-VM-Setup-Sample%2Fmain%2Fazuredeploy.json)

It takes time to complete all the provisioning. Once it's done, access to VM through either [RDP][az vm rdp] or [Bastion][az vm bastion].

![][image-01]

You can see all the applications have been installed!

---

So far, we've discussed how to automatically install applications for live streaming, using [Chocolatey][chocolatey], while provisioning a [Windows VM][az vm] on Azure. There are many reasons to provision and destroy VMs on the Cloud. Using an ARM Template and custom script for the VM provisioning will make your life easier. I hope this post gives small tips to live streamers using VMs for their purpose.

I'm going to discuss, in the [next post][post next], how [Power Platform][pw platform] increases productivity against this sort of ad-hoc Azure resource provisioning.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/app-provisioning-on-azure-vm-with-chocolatey-for-live-streaming-01.png

[post next]: /2020/09/02/ad-hoc-azure-resource-provisioning-via-power-platform/

[gh sample]: https://github.com/devkimchi/LiveStream-VM-Setup-Sample
[gh sample arm]: https://github.com/devkimchi/LiveStream-VM-Setup-Sample/blob/main/azuredeploy.json#L347-L389

[chocolatey]: https://chocolatey.org/

[henk tw]: https://twitter.com/hboelman
[henk blog]: https://www.henkboelman.com/articles/online-meetups-with-obs-and-skype/
[frank tw]: https://twitter.com/fboucheros
[frank blog]: http://www.frankysnotes.com/2018/04/dont-install-your-software-yourself.html

[ms edge]: https://www.microsoft.com/edge?WT.mc_id=devkimchicom-blog-juyoo
[skype]: https://www.skype.com/en/content-creators/
[obs]: https://obsproject.com/
[obs ndi]: https://obsproject.com/forum/threads/obs-ndi-newtek-ndi%E2%84%A2-integration-into-obs-studio.69240/
[ndi]: https://www.ndi.tv/

[az arm]: https://docs.microsoft.com/azure/azure-resource-manager/templates/overview?WT.mc_id=devkimchicom-blog-juyoo
[az quickstart]: https://azure.microsoft.com/resources/templates/?term=Deploy+a+simple+Windows+VM&WT.mc_id=devkimchicom-blog-juyoo
[az quickstart vm]: https://azure.microsoft.com/resources/templates/101-vm-simple-windows/?WT.mc_id=devkimchicom-blog-juyoo

[az vm]: https://docs.microsoft.com/azure/virtual-machines/windows/overview?WT.mc_id=devkimchicom-blog-juyoo
[az vm size]: https://docs.microsoft.com/azure/virtual-machines/sizes?WT.mc_id=devkimchicom-blog-juyoo
[az vm image]: https://docs.microsoft.com/azure/virtual-machines/windows/cli-ps-findimage?WT.mc_id=devkimchicom-blog-juyoo
[az vm custom script]: https://docs.microsoft.com/azure/virtual-machines/extensions/custom-script-windows?WT.mc_id=devkimchicom-blog-juyoo
[az vm rdp]: https://docs.microsoft.com/azure/virtual-machines/windows/connect-logon?WT.mc_id=devkimchicom-blog-juyoo
[az vm bastion]: https://docs.microsoft.com/azure/bastion/bastion-connect-vm-rdp?WT.mc_id=devkimchicom-blog-juyoo

[az storage blob]: https://docs.microsoft.com/azure/storage/blobs/storage-blobs-overview?WT.mc_id=devkimchicom-blog-juyoo

[pw platform]: https://powerplatform.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo
