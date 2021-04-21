---
title: "Azure Bicep Refreshed"
slug: bicep-refreshed
description: "This post refreshes the new features of Azure Bicep since the last update."
date: "2021-04-21"
author: Justin-Yoo
tags:
- azure
- bicep
- arm
- update
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/04/bicep-refreshed-00.png
fullscreen: true
---

In the [previous post][post 1], I introduced the very early stage of [Project Bicep][bicep]. At that time, it was the version of `0.1.x`, but now it's updated to `0.3.x`. You can use it for production, and many features keep being introduced. Throughout this post, I'm going to discuss new features added since the last post.

* [Project Bicep Sneak Peek][post 1]
* [GitHub Actions and ARM Template Toolkit for Bicep Codes Linting][post 2]
* ***Azure Bicep Refreshed***


## Azure CLI Integration ##

While Bicep CLI works as a stand-alone tool, it's been integrated with [Azure CLI][az cli] from [v2.20.0][az cli release v2.20.0] and later. Therefore, you can run bicep in either way.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=01-bicep-build.sh

> **NTOE**: Although Bicep CLI could build multiple files by `v0.2.x`, it's now only able to build one file at a time from `v0.3.x`. Therefore, if you want to build multiple files, you should do it differently. Here's a sample PowerShell script, for example.
> 
> https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=02-bicep-build.ps1

Because of the Azure CLI integration, you can also provision resources through the bicep file like below:

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=03-bicep-deploy.sh


## Bicep Decompiling ##

From [`v0.2.59`][bicep release v0.2.59], Bicep CLI can convert ARM templates to bicep files. It's particularly important because still many ARM templates out there have been running and need maintenance. Run the following command for decompiling.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=04-bicep-decompile.sh

> **NOTE**: If your ARM template contains a `copy` attribute, bicep can't decompile it as of this writing. In the later version, it should be possible.


## Decorators on Parameters ##

Writing parameters has become more articulate than `v0.1.x`, using the decorators. For example, there are only several possible SKU values of a Storage Account available, so using the `@allowed` decorator like below makes the code better readability.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=05-bicep-decorators.bicep


## Conditional Resources ##

You can use ternary operations for attributes. What if you can conditionally declare a resource itself using conditions? Let's have a look. The following code says if the location is only **Korea Central**, the Azure App Service resource can be provisioned.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=06-bicep-conditions.bicep


## Loops ##

While ARM templates use both `copy` attribute and `copyIndex()` function for iterations, bicep uses the `for...in` loop. Have a look at the code below. You can declare Azure App Service instances using the array parameter through the `for...in` loop.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=07-bicep-loops-1.bicep

You can also use both array and index at the same time.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=08-bicep-loops-2.bicep

Instead of the array, you can use the `range()` function in the loop.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=09-bicep-loops-3.bicep

Please note that you MUST use the array expression (`[...]`) outside the `for...in` loop because it declares the array of the resources. Bicep will do the rest.


## Modules ##

Personally, I love this part. While ARM templates use the [linked template][az arm template linked], bicep uses the `module` keyword for modularisation. Here's the example for Azure Function app provisioning. For this, you need at least Storage Account, Consumption Plan and Azure Functions resources. Each resource can be individually declared as a module, and the orchestration bicep file calls each module. Each module should work independently, of course.


### Storage Account ###

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=10-bicep-storage-account.bicep


### Consumption Plan ###

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=11-bicep-consumption-plan.bicep


### Azure Functions ###

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=12-bicep-function-app.bicep


### Modules Orchestration ###

Here's the orchestration bicep file to combine modules. All you need to do is to declare a module, refer to the module location and pass parameters. Based on the references between modules, dependencies are automatically calculated.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=13-bicep-azuredeploy.bicep

> **NOTE**: Unfortunately, as of this writing, referencing to external URL is not supported yet, unlike linked ARM templates.

---

So far, we've briefly looked at the new features of [Project Bicep][bicep]. As Bicep is one of the most rapidly growing toolsets in Azure, keep using it for your resource management.


[post 1]: /2020/09/09/bicep-sneak-peek/
[post 2]: /2020/09/30/github-actions-and-arm-template-toolkit-to-test-bicep-codes/

[bicep]: https://github.com/Azure/bicep/
[bicep release v0.2.59]: https://github.com/Azure/bicep/releases/tag/v0.2.59

[az cli]: https://docs.microsoft.com/cli/azure/what-is-azure-cli?WT.mc_id=devops-25381-juyoo
[az cli release v2.20.0]: https://docs.microsoft.com/cli/azure/release-notes-azure-cli?WT.mc_id=devops-25381-juyoo#march-02-2021

[az arm template linked]: https://docs.microsoft.com/azure/azure-resource-manager/templates/linked-templates?WT.mc_id=devops-25381-juyoo
