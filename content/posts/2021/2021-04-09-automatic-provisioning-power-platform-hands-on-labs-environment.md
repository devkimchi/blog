---
title: "Automatic Provisioning for Power Platform Hands-on Labs Environment"
slug: automatic-provisioning-power-platform-hands-on-labs-environment
description: "This post discusses how to automate environment provisioning for Power Platform hands-on labs for session leaders."
date: "2021-04-09"
author: Justin-Yoo
tags:
- power-platform
- microsoft365
- provisioning
- powershell
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/04/automatic-provisioning-power-platform-hands-on-labs-environment-00.png
fullscreen: true
---

Suppose you are a community leader or an instructor who will run a hands-on lab session for [Power Platform][pp]. You got content for it. Now it's time for setting up the lab environment. There are roughly three approaches for the preparation.

1. Ask the participants to bring their existing Power Platform environment,
1. Ask the participants to set up their environment by themselves, or
1. The session leader is preparing the environment for the participants to use.

Each effort has its pros and cons like:

1. **The first approach** would be the easiest and the most convenient for the instructor because it's based on the assumption that everyone is ready for the exercise. However, you never know if every participant has the same configurations as you expect. It really depends on their organisation's policy. After all, you, as the session leader, will probably suffer from a lot of unexpected circumstances.
2. **The second one** can be convenient for you as the session leader. It might be as tricky as the first approach. Delegating the environment set-up efforts to the participants may make you free, but at the same time, you should provide an instructional document very thoroughly and carefully. Even if you do so, it entirely depends on the participants' capability. After all, you should start the lab session by confirming the environment set-up anyway.
3. **The last option** goes to you as the session leader. You prepare everything for the participants. They just come, sit and practice. If you do this set-up by hand, it would be awful. You will not want to do that.

Therefore, as a hands-on lab session leader, I'm going to discuss how to automate all the provisioning process and minimise human intervention by running one PowerShell script.

> The PowerShell script used in this post is downloadable from [this GitHub repository][gh sample].


## Create Microsoft 365 Tenant ##

The first step to do as the session leader is to create a [Microsoft 365][m365] tenant. Microsoft 365 offers a free trial for 30 days. It includes 25 seats, including the admin account, which is suitable for the lab. Click this link, [http://aka.ms/Office365E5Trial][m365 trial e5], and create the Microsoft 365 E5 plan's trial tenant.

![Microsoft 365 E5 Trial Landing Page][image-01]

After filling out the form below, you get the trial tenant!

![Microsoft 365 E5 Trial Sign-up Page][image-02]

> Let's say you use the following information for the admin account.
> 
> * Tenant Name: `powerplatformhandsonlab`
> * Tenant URL: `powerplatformhandsonlab.onmicrosoft.com`
> * Admin E-mail: `admin@powerplatformhandsonlab.onmicrosoft.com`
> * Admin Password: `Pa$$W0rd!@#$`

As you've got a new tenant, let's configure the lab environment in PowerShell. **Please note that you HAVE TO use the PowerShell console with the admin privilege**.


## Install AzureAD Module ##

> **NOTE**: To use any of the PowerShell module mentioned in this post, you need PowerShell v5.1 running on Windows. PowerShell Core (v6 and later) doesn't support this scenario. For more details about this, refer to this page, [Connect to Microsoft 365 with PowerShell][m365 powershell connect].

You can add a new user account to a Microsoft 365 tenant through the [`AzureAD` module][psgallery azuread]. As of this writing, the latest version of the module is `2.0.2.130`. Use the `Install-Module` cmdlet to install the module. If you append these two parameters, `-Force -AllowClobber` (line #3), it always installs the newest version regardless it's already installed or not.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=01-install-azuread.ps1&highlights=3


## Log-in to AzureAD as Admin ##

After installing the module, log into the Azure AD as the tenant admin. For automation, you should stay within the console. Therefore, the following command is more efficient for sign-in.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=02-connect-azuread.ps1


## Add User Accounts ##

It's time to add user accounts. As the trial tenant includes 25 licenses, you can add up to 24 accounts. For more details to add a new user account, refer to this document, [Create Microsoft 365 User Accounts with PowerShell][m365 powershell account create]. But you just run the following commands. Here are some assumptions:

* Each user has the same password of `UserPa$$W0rd!@#$` for convenience, and it's not allowed change (line #2-4).
* Each user has the same location where the tenant resides. For now, it's `KR` (line #6).
* You need to create up to 24 accounts, so `ForEach-Object` is the go (line #9).
* All user accounts created are added to the `$users` array object (line #18).

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=03-new-azureaduser.ps1&highlights=2-4,6,9,18


## Assign Microsoft 365 Roles to User Accounts ##

The user accounts need to have appropriate Microsoft 365 roles. As it's the hands-on lab configuration, you can assign the Power Platform admin role to each user account. For more details of the Microsoft roles assignment, refer to this [Assign Admin Roles to Microsoft 365 User Accounts with PowerShell][m365 powershell role assign] page. Run the following command to activate the Power Platform admin role.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=04-get-azureaddirectoryrole.ps1

The admin role has now been stored in the `$role` object. Now, iterate the `$users` array to assign the role.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=05-add-azureaddirectoryrolemember.ps1


## Assign License to User Accounts ##

To use Power Platform within the tenant, each user MUST have a license for it. You can assign the license through the PowerShell command. For more details, visit this [Assign Microsoft 365 licenses to user accounts with PowerShell][m365 powershell license assign] page.

First of all, let's find out the licenses. As soon as you create the trial tenant, there SHOULD be only one license, whose name is `ENTERPRISEPREMIUM`.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=06-get-azureadsubscribedsku.ps1

Then, run the following command to assign the license to all users by iterating the `$users` array.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=07-set-azureaduserlicense.ps1

So far, you've completed automating processes to create a trial tenant, create user accounts, and assign roles and licenses.


## Activate Microsoft Dataverse for Power Platform Default Environment ##

Power Platform internally uses [Microsoft Dataverse][pp dataverse] as its database. Microsoft Dataverse is fundamentally essential for other Microsoft 365 services to use. You can also initialise it through PowerShell commands. For more details, visit the [Power Apps Cmdlets for Administrators][pp powershell dataverse create] page.

First, you need to install both PowerShell modules, [Microsoft.PowerApps.Administration.PowerShell][psgallery pa admin] and [Microsoft.PowerApps.PowerShell][psgallery pa]. Like the previous installation process, use the `-Force -AllowClobber` option to install the modules or reinstall both if they already exist (line #3, 7).

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=08-install-powerapps.ps1&highlights=3,7

Log into the Power Apps admin environment, using `$adminUpn` and `$adminPW` values.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=09-add-powerappsaccount.ps1

> **NOTE**: You might not be able to log into the Power Apps admin environment with the following error.
> 
> ![Unable to Login to Power Apps Environment][image-03]
> 
> It's because the internal log-in process for both Microsoft 365 tenant and Power Apps environment are different from each other. If it happens to you, don't panic. Just open a new PowerShell console with an admin privilege and attempt to log in.

Here are some assumptions for the Microsoft Dataverse initialisation:

* Initialise Microsoft Dataverse on the default environment (line #1),
* Follow the currency settings of the default environment (line #5), and
* Follow the language settings of the default environment (line #10).

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=10-new-adminpowerappcdsdatabase.ps1&highlights=1,5,10


## Assign Azure Subscription ##

Building [custom connectors][pp cuscon] is inevitable while using Power Platform. In this case, you might need to handle resources on Azure, which requires an Azure subscription. If you create the trial tenant for Microsoft 365, you can also activate the trial Azure subscription. As it requires credit card verification, it MUST be done within Azure Portal. If you log into Azure Portal with your admin account, you can see the following screen.

![Azure Subscription Trial Page][image-04]

Click the **Start** button to sign-up for the trial subscription.

![Azure Subscription Trial Sign-up Page][image-05]

Once completing the trial subscription, log in to Azure using the PowerShell command below. The `$adminCredential` object is the same one used for Azure AD log-in.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=11-connect-azaccount.ps1

> **NOTE**: You SHOULD install the [Az][psgallery az] module beforehand.
> 
> https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=12-install-az.ps1

Only a limited number of resources are available in the trial subscription. For custom connectors, mainly [Azure Logic Apps][az logapp], [Asture Storage Account][az st], [Azure Virtual Network][az netwrk], [Azure API Management][az apim] and [Azure Cosmos DB][az cosdba] are supposed to use. Therefore, to use those resources, run the following command to register those resource providers.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=13-register-azresourceprovider.ps1

Then, assign the subscription to each user account. For Azure Roles, visit this [Assign Azure Roles Using Azure PowerShell][az powershell role assign] page for more details.

> **NOTE**: Instead of scoping the entire subscription to each user account, it's better to create a resource group for each user, scope to the resource group and assign it to each account. For the resource group, you need a location. In this example, `koreacentral` is used.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=14-new-azroleassignment.ps1

All users are now able to access to Azure resources for the exercise.

---

So far, we've walked through how to automatically provision a Power Platform environment for hands-on-labs, using PowerShell. Now, if you are going to run a hands-on lab session and need a new environment, simply run the code above. Then, it's all good to go!


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/04/automatic-provisioning-power-platform-hands-on-labs-environment-01-en.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/04/automatic-provisioning-power-platform-hands-on-labs-environment-02-en.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/04/automatic-provisioning-power-platform-hands-on-labs-environment-03-en.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/04/automatic-provisioning-power-platform-hands-on-labs-environment-04-en.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/04/automatic-provisioning-power-platform-hands-on-labs-environment-05-en.png

[gh sample]: https://github.com/devkimchi/PowerPlatform-Hands-on-Lab-Environment-Automatic-Provsioning

[pp]: https://powerplatform.microsoft.com/?WT.mc_id=power-23654-juyoo
[pp dataverse]: https://docs.microsoft.com/powerapps/maker/data-platform/data-platform-intro?WT.mc_id=power-23654-juyoo
[pp cuscon]: https://docs.microsoft.com/connectors/custom-connectors/?WT.mc_id=power-23654-juyoo

[pp powershell dataverse create]: https://docs.microsoft.com/power-platform/admin/powerapps-powershell?WT.mc_id=power-23654-juyoo#power-apps-cmdlets-for-administrators

[psgallery azuread]: https://www.powershellgallery.com/packages/AzureAD/
[psgallery pa]: https://www.powershellgallery.com/packages/Microsoft.PowerApps.PowerShell/
[psgallery pa admin]: https://www.powershellgallery.com/packages/Microsoft.PowerApps.Administration.PowerShell/
[psgallery az]: https://www.powershellgallery.com/packages/Az/

[m365]: https://www.microsoft.com/microsoft-365?WT.mc_id=power-23654-juyoo
[m365 trial e5]: http://aka.ms/Office365E5Trial

[m365 powershell connect]: https://docs.microsoft.com/microsoft-365/enterprise/connect-to-microsoft-365-powershell?WT.mc_id=power-23654-juyoo
[m365 powershell account create]: https://docs.microsoft.com/microsoft-365/enterprise/create-user-accounts-with-microsoft-365-powershell?WT.mc_id=power-23654-juyoo
[m365 powershell role assign]: https://docs.microsoft.com/microsoft-365/enterprise/assign-roles-to-user-accounts-with-microsoft-365-powershell?WT.mc_id=power-23654-juyoo
[m365 powershell license assign]: https://docs.microsoft.com/microsoft-365/enterprise/assign-licenses-to-user-accounts-with-microsoft-365-powershell?WT.mc_id=power-23654-juyoo

[az powershell role assign]: https://docs.microsoft.com/azure/role-based-access-control/role-assignments-powershell?WT.mc_id=power-23654-juyoo

[az logapp]: https://docs.microsoft.com/azure/logic-apps/logic-apps-overview?WT.mc_id=power-23654-juyoo
[az st]: https://docs.microsoft.com/azure/storage/?WT.mc_id=power-23654-juyoo
[az netwrk]: https://docs.microsoft.com/azure/virtual-network/virtual-networks-overview?WT.mc_id=power-23654-juyoo
[az apim]: https://docs.microsoft.com/azure/api-management/api-management-key-concepts?WT.mc_id=power-23654-juyoo
[az cosdba]: https://docs.microsoft.com/azure/cosmos-db/introduction?WT.mc_id=power-23654-juyoo
