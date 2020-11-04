---
title: "Deploying Azure Functions via GitHub Actions without Publish Profile"
slug: deploying-azure-functions-via-github-actions-without-publish-profile
description: "This post shows how to deploy Azure Functions app via GitHub Actions, without having to know the publish profile."
date: "2020-11-05"
author: Justin-Yoo
tags:
- azure-functions
- github-actions
- azure
- publish-profile
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2020/11/deploying-azure-functions-via-github-actions-without-publish-profile-00.png
fullscreen: true
---

Throughout this series, I'm going to show how an Azure Functions instance can map APEX domains, add an SSL certificate and update its public inbound IP address to DNS.

* [3 Ways Mapping APEX Domains to Azure Functions][post 1]
* [Adding Let's Encrypt SSL Certificate to Azure Functions][post 2]
* [Updating Azure DNS and SSL Certificate on Azure Functions via GitHub Actions][post 3]
* ***Deploying Azure Functions Apps via GitHub Actions without Publish Profile***

In my [previous post][post 3], I walked through how to update an A record of DNS server and renew the SSL certificate automatically, when an inbound IP of the [Azure Functions][az func] instance changes, using [GitHub Actions][gh actions] workflow. As the last post of this series, I'm going to discuss how to deploy the Azure Functions app through GitHub Actions workflow, without having to know the publish profile.


## Azure Functions Action ##

There is an official [GitHub Actions for Azure Functions deployment][gh actions azfunc] on GitHub Marketplace. The following YAML pipeline shows how to use it. The `publish-profile` parameter takes the publish profile of your Azure Functions instance for deployment (line #11).

https://gist.github.com/justinyoo/d806e47efde7749c6154b19f081f86d2?file=01-az-func-deploy.yaml&highlights=11

As you can see the sample pipeline above, we should store the publish profile onto the repository's secrets. If the publish profile is reset for some reason, the secret MUST be updated with the new profile, **manually**. It's cumbersome. What if we populate and make use of the publish profile within the GitHub Actions workflow? Yes, we can!


## GitHub Action: PowerShell Scripts ##

In order to populate the publish profile, you need to log-in to [Azure PowerShell][az pwsh] through [Azure Login][gh actions az login]. The `enable-AzPSSession` parameter value of `true` lets you log-in to Azure PowerShell session (line #9).

https://gist.github.com/justinyoo/d806e47efde7749c6154b19f081f86d2?file=02-az-login.yaml&highlights=9

Then, get the publish profile value, using the PowerShell action below (line #10-12). As the publish profile is basically an XML document, you should remove all the line-feed characters to make the XML document into one linear string (line #14). Finally, the XML document is set to an output value (line #16).

https://gist.github.com/justinyoo/d806e47efde7749c6154b19f081f86d2?file=03-get-publish-profile.yaml&highlights=10-12,14,16

Once it's done, let's render the output value on the workflow log.

https://gist.github.com/justinyoo/d806e47efde7749c6154b19f081f86d2?file=04-show-publish-profile.yaml&highlights=12

The result might look like the following. Here's the weird thing. We got the publish profile successfully, but the password part is not properly masked. Therefore, as soon as the publish profile is displayed like this, we MUST assume that this publish profile is no longer safe to use.

![][image-01]

I mean, the publish profile itself is still valid. But after the deployment, it's safe to reset the profile from the security perspective. Therefore, use the following action to reset the publish profile (line #9-13).

https://gist.github.com/justinyoo/d806e47efde7749c6154b19f081f86d2?file=05-reset-publish-profile.yaml&highlights=9-13

So, the entire workflow for Azure Functions deployment is:

1. Download the publish profile for the Azure Functions app,
2. Deploy the Functions app using the publish profile, and
3. Reset the publish profile for the Azure Functions app.


## GitHub Action: Azure App Service Publish Profile ##

Those PowerShell script needs to be written every time you define a new workflow. But it would be great if there is GitHub Actions for it. Of course, there is. If you use this [Azure App Service Publish Profile][gh actions appsvc profile] action, you can get the publish profile and reset it easily. Let's set up the workflow like below:

1. Download the publish profile for the Azure Functions app (line #12-19),
2. Deploy the Functions app using the publish profile (line #26), and
3. Reset the publish profile for the Azure Functions app (line #28-35).

https://gist.github.com/justinyoo/d806e47efde7749c6154b19f081f86d2?file=06-workflow.yaml&highlights=12-19,26,28-35

With this action, we don't have to know the publish profile, but the workflow takes care of it. In addition to that, by any chance the publish profile is compromised, the last action always resets the profile. Therefore, the compromised one is no longer valid.

---

So far, we've walked through how we can deploy Azure Functions app through [GitHub Actions][gh actions], with no knowledge of the publish profile and reset the profile. I hope this approach would help build your CI/CD pipeline.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/11/deploying-azure-functions-via-github-actions-without-publish-profile-01.png

[post 1]: /2020/10/07/3-ways-mapping-apex-domains-to-azure-functions/
[post 2]: /2020/10/14/lets-encrypt-ssl-certificate-on-azure-functions/
[post 3]: /2020/10/28/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions/
[post 4]: /2020/11/05/deploying-azure-functions-via-github-actions-without-publish-profile/

[az func]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=devops-10695-juyoo

[az pwsh]: https://docs.microsoft.com/powershell/azure/new-azureps-module-az?WT.mc_id=devops-10695-juyoo

[gh actions]: https://docs.github.com/en/free-pro-team@latest/actions
[gh actions az login]: https://github.com/marketplace/actions/azure-login#configure-deployment-credentials
[gh actions azfunc]: https://github.com/marketplace/actions/azure-functions-action
[gh actions appsvc profile]: https://github.com/marketplace/actions/azure-app-service-publish-profile
