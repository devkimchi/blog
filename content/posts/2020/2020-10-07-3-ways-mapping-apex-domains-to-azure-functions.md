---
title: "3 Ways Mapping APEX Domains to Azure Functions"
slug: 3-ways-mapping-apex-domains-to-azure-functions
description: "This post discusses how to map APEX domains to Azure Functions instance in three different ways."
date: "2020-10-07"
author: Justin-Yoo
tags:
- azure-functions
- apex-domain
- azure-cli
- arm-template
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/3-ways-mapping-apex-domains-to-azure-functions-00.png
fullscreen: true
---

Throughout this series, I'm going to show how an Azure Functions instance can map APEX domains, bind an SSL certificate to the custom domain and automatically update its public inbound IP address to DNS server.

* ***3 Ways Mapping APEX Domains to Azure Functions***
* Adding Let's Encrypt SSL Certificate to Azure Functions Custom Domain
* Updating Public IP Address of Azure Functions to DNS Automatically

Let's say there is an [Azure Functions][az func] instance. One of your customers wants to add a custom domain to the Function app. As long as the custom domain is a sub-domain type like `api.contoso.com`, it shouldn't be an issue because CNAME mapping is supported out-of-the-box. But what if the customer wants to add an APEX domain?

> Both APEX domain and root domain point to the same thing like `contoso.com`.

![][image-01]

Adding the root domain through [Azure Portal][az portal] can't be accomplished with the message above. To add the APEX domain to Azure Functions instance, it requires an A record that needs a public IP address. But Azure Functions instance doesn't support it via the portal.

Should we give it up now? Well, not really.

![][image-02]

As always, there's a way to get around. Throughout this post, I'm going to show how to map the APEX domain to an Azure Functions instance in three different ways.


## Verifying Domain Ownership ##

First of all, you need to verify the domain ownership at the `Custom Domains` blade of the Function app instance.

![][image-03]

* Get the `Custom Domain Verification ID` at the picture above.
* Add the TXT record, `asuid.contoso.com`, to your DNS server.
* Add the A record with the IP address to your DNS server.

The verification process can be done via the portal.

> If you want to run the verification with [Azure CLI][az cli], please have a look at the [link][gh azcli].

![][image-04]

Now, you've verified the domain ownership. But you haven't still yet made the APEX domain mapping.


## 1. Through Azure PowerShell ##

If you can't make it through [Azure Portal][az portal], [Azure PowerShell][az pwsh] will be one alternative. Use the `Set-AzWebApp` cmdlet to map your APEX domain.

https://gist.github.com/justinyoo/8f4f33645adf7426969855b29171e91a?file=01-set-azwebapp.ps1&highlights=7

When you use Azure PowerShell, you MUST make sure one thing. The `-HostNames` parameter specified above MUST include the existing domain names (line #7). Otherwise, all existing domains will be removed, and you will get the warning like below:

![][image-05]

If you add all custom domains including the default domain name like `*.azurewebsites.net`, you will be able to see the screen below:

![][image-06]


## 2. Through Azure CLI ##

If you prefer to using [Azure CLI][az cli], use the command, `az functionapp config hostname add`.

https://gist.github.com/justinyoo/8f4f33645adf7426969855b29171e91a?file=02-az-functionapp.sh

Unlike [Azure PowerShell][az pwsh], you can add one hostname at a time and won't lose the ones already added.


## 3. Through ARM Template ##

You can also use ARM Template instead of command-line commands. Here's the [bicep][gh bicep] code for it (line #5-8).

https://gist.github.com/justinyoo/8f4f33645adf7426969855b29171e91a?file=03-add-hostname.bicep&highlights=5-8

When the bicep file is compiled to ARM Template, it looks like:

https://gist.github.com/justinyoo/8f4f33645adf7426969855b29171e91a?file=04-add-hostname.json

---

So far, we've used three different ways to map an APEX domain to [Azure Functions][az func] instance. Generally speaking, it's rare to map a custom domain to an Azure Functions instance. It's even rarer to map the APEX domain. Therefore, the Azure Portal doesn't support this feature. However, as we already saw, we can use either [Azure PowerShell][az pwsh] or [Azure CLI][az cli], or ARM templates to add the root domain. I hope this post helps if one of your clients' requests is the one described in this post.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/3-ways-mapping-apex-domains-to-azure-functions-01-en.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/3-ways-mapping-apex-domains-to-azure-functions-02-en.jpg
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/3-ways-mapping-apex-domains-to-azure-functions-03-en.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/3-ways-mapping-apex-domains-to-azure-functions-04-en.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/3-ways-mapping-apex-domains-to-azure-functions-05-en.png
[image-06]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/3-ways-mapping-apex-domains-to-azure-functions-06-en.png

[post 1]: /2020/10/07/tbp/
[post 2]: /2020/10/14/tbp/
[post 3]: /2020/10/21/tbp/

[az func]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=devkimchicom-blog-juyoo
[az portal]: https://azure.microsoft.com/features/azure-portal/?WT.mc_id=devkimchicom-blog-juyoo
[az cli]: https://docs.microsoft.com/cli/azure/what-is-azure-cli?WT.mc_id=devkimchicom-blog-juyoo
[az pwsh]: https://docs.microsoft.com/powershell/azure/new-azureps-module-az?WT.mc_id=devkimchicom-blog-juyoo

[gh azcli]: https://github.com/Azure/azure-cli/issues/14142#issuecomment-676539150
[gh bicep]: https://github.com/azure/bicep
