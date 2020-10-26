---
title: "Updating Azure DNS and SSL Certificate on Azure Functions via GitHub Actions"
slug: updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions
description: "This post shows how to automatically update A record on Azure DNS when the inbound IP address of Azure Functions instance is updated, and reflect the change to the SSL certificate through GitHub Actions."
date: "2020-10-28"
author: Justin-Yoo
tags:
- azure-functions
- azure-dns
- ssl-certificate
- github-actions
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-00.png
fullscreen: true
---

Throughout this series, I'm going to show how an Azure Functions instance can map APEX domains, add an SSL certificate and update its public inbound IP address to DNS.

* [3 Ways Mapping APEX Domains to Azure Functions][post 1]
* [Adding Let's Encrypt SSL Certificate to Azure Functions][post 2]
* ***Updating Azure DNS and SSL Certificate on Azure Functions via GitHub Actions***
* Deploying Azure Functions Apps via GitHub Actions without Publish Profile

In my [previous post][post 2], we walked through how to link an SSL certificate issued by [Let's Encrypt][letsencrypt], with a custom APEX domain. Throughout this post, I'm going to discuss how to automatically update the A record of a DNS server when the inbound IP address of the [Azure Functions][az func] instance is changed, update the SSL certificate through the [GitHub Actions][gh actions] workflow.

> All the GitHub Actions source codes used in this post can be found at [this repository][gh sample].


## Azure Functions Inbound IP Address ##

If you use an Azure Functions instance under [Consumption Plan][az func csplan], its inbound IP address is not static. In other words, the inbound IP address of an Azure Functions instance will be changing at any time, without prior notice. Actually, due to the serverless nature, we don't need to worry about the IP address change. If you see the instance details, it has more than one inbound IP address assignable.

![][image-01]

Therefore, if you map a custom APEX domain to your Azure Function instance, your APEX domain has to be mapped to an A record of your DNS. And whenever the inbound IP address changes, your DNS must update the A record as well.


## A Record Update on Azure DNS ##

If you use [Azure PowerShell][az pwsh], you can get the inbound IP address of your Azure Function app instance.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=01-get-inbound-ip-address.ps1

Then, check your DNS and find out the A record. Let's assume that you use [Azure DNS][az dns] service as your DNS. As there can be multiple A records registered, You'll take only the first one for now.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=02-get-arecord.ps1

Let's compare to each other. If the inbound IP and the A record are different, update the A record value of Azure DNS.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=03-compare-ip-address.ps1

We've got the A record update done.


## SSL Certificate Update on Azure Key Vault ##

If the A record has been updated, the existing SSL certificate is not valid any longer. Therefore, you should also update the SSL certificate. In my [previous post][post 2], I used the [SSL certificate update tool][gh acmebot keyvault], and it provides the [HTTP API endpoint][gh acmebot keyvault httpapi] to renew the certificate. Now, you can send the HTTP API request to the endpoint, through PowerShell.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=04-update-ssl-certificate.ps1

Now, you got the renewed SSL certificate by reflecting the updated A record.

![][image-02]

![][image-03]



## SSL Certificate Sync on Azure Functions ##

You got the SSL certificate renewed, but your Azure Function instance hasn't got the renewed certificate yet.

![][image-04]

According to the doc, the renewed SSL certificate will be automatically synced [in 48 hours][az kv certificate import]. If you think it's too long to take, use the following PowerShell script to sync the renewed certificate manually. First of all, get the access token from the login context. If you use the [Service Principal][az pwsh spn], you can get the access token by filtering the client ID of your Service Principal.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=05-get-access-token.ps1

Next, get the UNIX timestamp value in milliseconds.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=06-get-epoch.ps1

Then, make up the HTTP API endpoint to the certificate. As you've already logged in with your Service Principal, you already know the `$subscriptionId` value.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=07-set-endpoint.ps1

Call the endpoint to get the existing certificate details via the `GET` method.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=08-get-ssl-certificate.ps1

Call the same endpoint with the existing certificate details through the `PUT` method. Then, the renewed certificate is synced.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=09-sync-ssl-certificate.ps1

The `$result` object contains the result of the sync process. Both `$result.properties.thumbprint` value and `$cert.properties.thumbprint` value MUST be different. Otherwise, it's not synced yet. Once the sync process is over, you can find out the renewed thumbprint value on Azure Portal.

![][image-05]


## GitHub Actions Workflow for Automation ##

Now, we got three jobs for SSL certificate update. Let's build each job as a GitHub Action. By the way, why do I need GitHub Actions for this automation?

* GitHub Actions is not exactly the same, but it has the same nature of serverless &ndash; triggered by events and no need to set up the infrastructure.
* Unlike other serverless services, GitHub Actions doesn't need any infrastructure or instance setup or configuration because we only need a repository to run the GitHub Actions workflow.
* GitHub Actions is free of charge, as long as your repository is public (or open-source).

As all GitHub Actions are running Azure PowerShell scripts, we can simply define the common `Dockerfile`.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=10-az-powershell.dockerfile

The `entrypoint.ps1` file of each Action makes use of the logic stated above.


### A Record Update ###

This is the Action that updates A record on Azure DNS. It returns an output value indicating whether the A record has been updated or not. With this output value, we can decide whether to take further steps or not (line #27, 38).

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=11-update-arecord.ps1&highlights=27,38


### SSL Certificate Update ###

This is the Action that updates the SSL certificate on Azure Key Vault. It also returns an output value indicating whether the update is successful or not (line #14).

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=12-update-ssl-certificate.ps1&highlights=14


### SSL Certificate Sync ###

This is the Action that syncs the certificate on Azure Functions. It also returns an output indicating whether the sync is successful or not (line #44).

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=13-sync-ssl-certificate.ps1&highlights=44


### GitHub Actions Workflow ###

The ideal way to trigger the GitHub Actions workflow should be the event &ndash; when an inbound IP address changes on Azure Function instance, it should raise an event that triggers the GitHub Actions workflow. Unfortunately, at the time of this writing, there is no such event from Azure App Service. Therefore, you should use the [scheduler][gh actions schedule] instead. With the timer event of GitHub Actions, you can regularly check the inbound IP address change.

* As the scheduler is the main event trigger, set up the CRON scheduler (line #4-5). Here in the sample code, I run the scheduler for every 30 mins.
* As I use all the actions privately, not publicly, whenever the scheduler is triggered, checkout the code first (line #14-15).
* Update the A record of Azure DNS.
* Depending on the result of the A record update (line #29), it updates the SSL certificate.
* Depending on the result of the SSL certificate renewal (line #37), it syncs the SSL certificate with Azure Functions instance.
* Depending on the result of the SSL certificate sync (line #47), it sends a notification email to administrators.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=14-github-action-workflow.yaml&highlights=4-5,14-15,29,37,47

After the workflow runs, we can see the result like below:

![][image-06]

This is the notification email.

![][image-07]

If the A record is up-to-date, the workflow stops there and doesn't take the further steps.

![][image-08]

---

So far, we use [GitHub Actions workflow][gh actions] to regularly check the inbound IP address of the [Azure Functions][az func] instance and update the change to [Azure DNS][az dns], renew an SSL certificate, and sync the certificate with Azure Functions instance. In the next post, I'll discuss how to deploy the Azure Functions app through GitHub Actions without having to know the publish profile.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-04.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-05.png
[image-06]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-06.png
[image-07]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-07.png
[image-08]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-08.png

[post 1]: /2020/10/07/3-ways-mapping-apex-domains-to-azure-functions/
[post 2]: /2020/10/14/lets-encrypt-ssl-certificate-on-azure-functions/
[post 3]: /2020/10/28/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions/
[post 4]: /2020/10/28/tbp/

[az func]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=devops-10319-juyoo
[az func csplan]: https://docs.microsoft.com/azure/azure-functions/functions-scale?WT.mc_id=devops-10319-juyoo#consumption-plan

[az kv certificate import]: https://docs.microsoft.com/azure/app-service/configure-ssl-certificate?WT.mc_id=devops-10319-juyoo#import-a-certificate-from-key-vault
[az dns]: https://docs.microsoft.com/azure/dns/dns-overview?WT.mc_id=devops-10319-juyoo
[az pwsh]: https://docs.microsoft.com/powershell/azure/new-azureps-module-az?WT.mc_id=devops-10319-juyoo
[az pwsh spn]: https://docs.microsoft.com/powershell/azure/create-azure-service-principal-azureps?WT.mc_id=devops-10319-juyoo

[gh sample]: https://github.com/devrel-kr/dvrl-kr-dns-update
[gh actions]: https://docs.github.com/en/free-pro-team@latest/actions
[gh actions schedule]: https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows#schedule

[gh acmebot keyvault]: https://github.com/shibayan/keyvault-acmebot
[gh acmebot keyvault httpapi]: https://github.com/shibayan/keyvault-acmebot/wiki/REST-API

[letsencrypt]: https://letsencrypt.org/
