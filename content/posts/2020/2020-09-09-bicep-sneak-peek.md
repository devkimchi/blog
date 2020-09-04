---
title: "Project Bicep Sneak Peek"
slug: bicep-sneak-peek
description: "This post discusses how Bicep, the ARM template DSL, looks like and how we can leverage it for ARM template authoring."
date: "2020-09-09"
author: Justin-Yoo
tags:
- bicep
- azure-resource-manager
- arm-template
- sneak-peek
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2020/09/bicep-sneak-peek-00.png
fullscreen: true
---

Microsoft has recently revealed an [ARM Template][az arm template] DSL (Domain Specific Language), called [Bicep][gh bicep] to help devs build ARM templates quicker and easier.

Amongst several ways to provision resources onto [Azure][az], [ARM template][az arm template] is one popular approach for DevOps engineers. However, many DevOps engineers have been providing feedback that ARM template is hard to learn and deploy at scales, as it can be tricky. Therefore, field experts like Microsoft MVPs have suggested many best practices about authoring ARM templates, but it's still the big hurdle to get through.

As of this writing, it's v0.1, which is a very early preview. It means there will be many improvements until it becomes v1.0. Throughout this post, I'm going to discuss its expressions and how it can ease the ARM template authoring fatigues.

> The sample `.bicep` file used for this post can be fount at this [GitHub repository][gh sample].

**DO NOT USE BICEP ON YOUR PRODUCTION UNTIL IT GOES TO V0.3**


## ARM Template Skeleton Structure ##

ARM template has the basic structure like below:

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=01-arm-template.json

Those `parameters`, `variables`, `resources` and `outputs` attributes are as nearly as mandatory, so Bicep supports those attributes first. One of the problems that the ARM template structure currently has is the location to declare parameters, variables and resources. They MUST be sitting inside `parameters`, `variables` and `resources` sections, respectively, which is not as flexible as other programming languages. But Bicep has successfully sorted out this issue. Let's discuss this flexibility later on this post.


## Bicep Parameters ##

Parameters in Bicep can be declared like below. Every attribute is optional, by the way.

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=02-parameter-1.bicep

The simplest form of the parameters can be like below. Instead of putting the metadata for the parameter description within the parameter declaration, we can simply use the comments (line #1). Also, instead of setting the default value inside the parameter declaration, assign a default value as if we do it in any programming language (line #7). Of course, we don't have to set a default value to the parameter (line #8).

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=03-parameter-2.bicep&highlights=1,7,8

On the other hands, to use `secure` or `allowed` attribute, the parameter should be declared as an object form (line #3-5).

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=04-parameter-3.bicep&highlights=3-5

Note that the parameter declaration is all about what type of value we will accept from outside, not what the actual value will be. Therefore, it doesn't use the equal sign (`=`) for the parameter declaration, except assigning its default value.


## Bicep Variables ##

While the parameters accept values from outside, variables define the values to be used inside the template. The variables are defined like below. We can use all of existing ARM template functions to handle strings. But as Bicep supports string interpolations and ternary conditional operators, we can replace many [`concat()` function][az arm function concat]s and [`if()` function][az arm function if]s with them (line #2-3).

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=05-variable.bicep&highlights=2-3

Note that, unlike the parameters, the variables use the equal sign (`=`) because it assigns a value to the variable.


## Bicep Resources ##

Bicep declares resources like below.

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=06-resource.bicep&highlights=1,19

There are several things worth noting.

* The format to declare a resource is similar to the parameter. The parameter declaration looks like `param <identifier> <type>`. The resource declaration looks similar to `resource <identifier> <type>`.
* However, the resource type section is way different from the parameter declaration. It has a definite format of `<resource namespace>/<resource type>@<resource API version>` (line #1).
  * I prefer to using the [`providers()` function][az arm function providers] as I can't remember all the API versions for each resource.
  * But using the `providers()` function is [NOT recommended][az arm validation providers]. Instead, the API version should be explicitly declared. To find out the latest API version, use the following PowerShell command.

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=07-get-provider.ps1

* Bicep automatically resolves the dependencies between resources by using the resource identifier.
  * For example, the Storage Account resource is declared first as `st`, then `st` is referred within the Virtual Machine declaration (line #19).
  * On the other hands, in the ARM template world, we should explicitly declare resources within the `dependsOn` attribute to define dependencies.

You might have found an interesting fact while authoring Bicep file.

* ARM templates should rigorously follow the schema to declare parameters, variables and resources. Outside its respective section, we can't declare them.
* On the other hands, Bicep is much more flexible with regards to the location where to declare parameters, variables and resources.
  * We can simply declare `param`, `var` and `resource` wherever you like, within the Bicep file, then it's automagically sorted out during the build time.


## Bicep Installation and Usage ##

As mentioned above, Bicep has just started its journey, and its version is v0.1, meaning there are a lot of spaces for improvements. Therefore, it doesn't have a straightforward installation process. But follow the [installation guide][az bicep install], and you'll be able to make it. Once the installation completes, run the following command to build the Bicep file.

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=08-build-bicep.sh


## Rough Comparison to ARM Template Authoring ##

Let's compare the result between the [original ARM template][az arm template manual] and [Bicep-built ARM template][az arm template bicep]. Both work the same, but the original file has 415 lines of code while Bicep-generated one cuts to 306 lines of the code. In addition to this, Bicep file itself is even as short as 288 lines of code. If I refactor more by simplifying all the variables, the size of the Bicep file will be much shorter.

---

So far, we have had a quick look at the early preview of the [Bicep][gh bicep] project. It was pretty impressive from the usability point of view and has improved developer experiences way better. Please play around this tool and feel the difference by authoring the Bicep file.


[gh sample]: https://github.com/devkimchi/LiveStream-VM-Setup-Sample/blob/main/bicep/azuredeploy.bicep
[gh bicep]: https://github.com/Azure/bicep

[az]: https://azure.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo

[az arm template]: https://docs.microsoft.com/azure/azure-resource-manager/templates/overview?WT.mc_id=devkimchicom-blog-juyoo
[az arm template manual]: https://github.com/devkimchi/LiveStream-VM-Setup-Sample/blob/main/azuredeploy.json
[az arm template bicep]: https://github.com/devkimchi/LiveStream-VM-Setup-Sample/blob/main/bicep/azuredeploy.json
[az arm function concat]: https://docs.microsoft.com/azure/azure-resource-manager/templates/template-functions-string?WT.mc_id=devkimchicom-blog-juyoo#concat
[az arm function if]: https://docs.microsoft.com/azure/azure-resource-manager/templates/template-functions-logical?WT.mc_id=devkimchicom-blog-juyoo#if
[az arm function providers]: https://docs.microsoft.com/azure/azure-resource-manager/templates/template-functions-resource?WT.mc_id=devkimchicom-blog-juyoo#providers
[az arm validation providers]: https://docs.microsoft.com/azure/azure-resource-manager/templates/test-cases?WT.mc_id=devkimchicom-blog-juyoo#use-hardcoded-api-version

[az bicep install]: https://github.com/Azure/bicep/blob/master/docs/installing.md
