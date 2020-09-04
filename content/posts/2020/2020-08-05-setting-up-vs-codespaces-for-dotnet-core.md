---
title: "Setting-up Visual Studio Codespaces for .NET Core"
slug: setting-up-vs-codespaces-for-dotnet-core
description: "This post shows how to set up a common development environment for Visual Studio Codespaces to build .NET Core applications."
date: "2020-08-05"
author: Justin-Yoo
tags:
- visual-studio-codespaces
- github-codespaces
- environment-setup
- dotnet-core
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/setting-up-vs-codespaces-for-dotnet-core-00.png
fullscreen: true
---

Since April 2020 [Visual Studio Codespaces has been generally available][vs cs]. In addition to that, [GitHub Codespaces][gh cs] has been provided as a private preview. Both are very similar to each other in terms of their usage. There are differences between both, though, discussed from [this post][devto post]. Throughout this post, I'm going to focus on the .NET Core application development.

[Visual Studio Codespaces (VS CS)][vs cs] is an online IDE service running on a VM somewhere in [Azure][az]. Like Azure DevOps build agents, this VM is provisioned when a [VS CS][vs cs] begins and destroyed after the [VS CS][vs cs] is closed. With no other configuration, it is provisioned with default values. However, it's not enough for .NET Core application development. Therefore, we might have to add some configurations.

What if, there is a pre-configured .NET Core development environment ready? It can be sorted out in two different ways. One is [configuring the dev environment for each project or repository][vs cs config], and the other is [personalising the dev environment][vs cs personal]. The former approach would be better to secure a common ground for the entire project, while each individual in the team can personalise their own environment using the latter approach. This post focuses on the former one.


## Configuring Development Environment ##

As a Docker container takes care of the dev environment, we should define the `Dockerfile`. As there's already a [pre-configured one][gh vs cs config], we simply use it. But let's build our opinionated one! There are roughly two parts &ndash; Docker container and extensions.

> The sample environment can be found at [this repository][gh sample].


### Create `.devcontainer` Directory ###

First of all, we need to create the `.devcontainer` directory within the repository. This directory contains `Dockerfile`, a bash script that the Docker container executes, and `devcontainer.json` that defines extensions.


### Define `Dockerfile` ###

As there's an official Docker image for .NET Core SDK, we just use it as a base image. Here's the `Dockerfile`. The [`3.1-bionic`][docker hub dotnet core] tag is for Ubuntu 18.04 LTS (line #1). If you want to use a different Linux distro, choose a different tag.

https://gist.github.com/justinyoo/491cd606bd3b646f3fd5773d104e46f5?file=01-dockerfile.txt&highlights=1

Now, let's move on for `setup.sh`.


### Configure `setup.sh` ###

In the `setup.sh`, we're going to install several applications:

1. Update the existing packages through the `apt-get` command. If there's a new package, the script will install the new ones. (line #1-9).
2. Install `nvm` for ASP.NET Core application development, which uses `node.js` (line #12).
3. The Docker image as on today includes PowerShell 7.0.2. If you want to install the latest version of PowerShell, run this part (line #15).
4. If you want to use zsh instead of bash, [oh my zsh][gh ohmyzsh] enhances developer experiences (line #18-22).

https://gist.github.com/justinyoo/491cd606bd3b646f3fd5773d104e46f5?file=02-setup.sh&highlights=1-9,12,15,18-22

We now have both `Dockerfile` and `setup.sh` for the container setting. It's time to install extensions for [VS CS][vs cs] to use for .NET Core application development.


### List of Extensions ###

`devcontainer.json` is the entry point when a new [VS CS][vs cs] instance is firing up. The `Dockerfile` we defined above is linked to this `devcontainer.json` so that the development environment is provisioned. Through this `devcontainer.json` file, we can install all the necessary extensions to use. I'm going to install the following extensions for .NET Core app development.

* [Azure Tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-node-azure-pack&WT.mc_id=devkimchicom-blog-juyoo)
* [Bracket Pair Colorizer 2](https://marketplace.visualstudio.com/items?itemName=CoenraadS.bracket-pair-colorizer-2&WT.mc_id=devkimchicom-blog-juyoo)
* [C#](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp&WT.mc_id=devkimchicom-blog-juyoo)
* [C# Extensions](https://marketplace.visualstudio.com/items?itemName=kreativ-software.csharpextensions&WT.mc_id=devkimchicom-blog-juyoo)
* [C# Sort Usings](https://marketplace.visualstudio.com/items?itemName=jongrant.csharpsortusings&WT.mc_id=devkimchicom-blog-juyoo)
* [C# XML Documentation Comments](https://marketplace.visualstudio.com/items?itemName=k--kato.docomment&WT.mc_id=devkimchicom-blog-juyoo)
* [Docs Authoring Pack](https://marketplace.visualstudio.com/items?itemName=docsmsft.docs-authoring-pack&WT.mc_id=devkimchicom-blog-juyoo)
* [EditorConfig](https://marketplace.visualstudio.com/items?itemName=EditorConfig.EditorConfig&WT.mc_id=devkimchicom-blog-juyoo)
* [Git Graph](https://marketplace.visualstudio.com/items?itemName=mhutchie.git-graph&WT.mc_id=devkimchicom-blog-juyoo)
* [Git History](https://marketplace.visualstudio.com/items?itemName=donjayamanne.githistory&WT.mc_id=devkimchicom-blog-juyoo)
* [GitHub Pull Requests and Issues](https://marketplace.visualstudio.com/items?itemName=github.vscode-pull-request-github&WT.mc_id=devkimchicom-blog-juyoo)
* [GitLens](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens&WT.mc_id=devkimchicom-blog-juyoo)
* [IntelliCode](https://marketplace.visualstudio.com/items?itemName=visualstudioexptteam.vscodeintellicode&WT.mc_id=devkimchicom-blog-juyoo)
* [Live Share](https://marketplace.visualstudio.com/items?itemName=ms-vsliveshare.vsliveshare&WT.mc_id=devkimchicom-blog-juyoo)
* [Markdown All in One](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one&WT.mc_id=devkimchicom-blog-juyoo)
* [PowerShell](https://marketplace.visualstudio.com/items?itemName=ms-vscode.PowerShell&WT.mc_id=devkimchicom-blog-juyoo)
* [vscode-icons](https://marketplace.visualstudio.com/items?itemName=vscode-icons-team.vscode-icons&WT.mc_id=devkimchicom-blog-juyoo)

Define those extensions in the `devcontainer.json` like:

https://gist.github.com/justinyoo/491cd606bd3b646f3fd5773d104e46f5?file=03-devcontainer.json


### What Else Does `devcontainer.json` Do? ###

We've just defined all the extensions in the `devcontainer.json` file. What else does this file do? It configures overall environments for [VS CS][vs cs] to use. It's OK to leave as default, but if you really want to configure, please refer to this [official document][vs cs config]. Here in this post, I'll pick up a few points.

* `dockerFile`: Set the value as `Dockerfile` that we defined above.
* `forwardPorts`: When [VS CS][vs cs] takes some ports, they need to be forwarded so that we can debug that on our web browser. For example, ASP.NET Core application needs both `5000` and `5001` ports, and Azure Functions takes `7071`. Put these ports into an array and assign it to this attribute.
* `settings`: It's to configure the [VS CS][vs cs] editor settings.


All the environment setup is done!


## Run GitHub Codespaces ##

Push this `.devcontainer` directory back to the repository and run [GitHub Codespaces][gh cs]. If you've already joined in the GitHub Codespaces Early Access Program, you'll be able to see the menu like below:

![][image-01]

Click the menu, and you'll be able to access to all files within [GitHub Codespaces][gh cs].

![][image-02]

In addition to that, we have both a PowerShell console and zsh console. Run the following command to create a sample .NET Core console app to start writing the code!

https://gist.github.com/justinyoo/491cd606bd3b646f3fd5773d104e46f5?file=04-dotnet-new.sh

The `Program.cs` should be looking like this!

![][image-03]


## Run Visual Studio Codespaces ##

This time, run the same repository on [VS CS][vs cs]. First of all, visit [https://online.visualstudio.com][vs cs vso] and login.

> You **MUST** have an active Azure subscription to run a [VS CS][vs cs] instance. If you don't, create a free account and subscription through this [Free Azure account][az free] page.

After the login, unless you have a billing plan, you should create it. [VS CS][vs cs] takes the consumption-based model, which means that [you only have to pay for what you have used][az vso pricing]. If you don't need it any longer, delete it to avoid the further charge.

![][image-04]

You will be asked to create a new instance if you don't have one yet.

![][image-05]

Enter the requested information and create a new [VS CS][vs cs] instance. The screenshot below links the GitHub repository, which is dedicated to the repository. If you don't link any of GitHub repository, it can be used for any repository.

![][image-06]

The [VS CS][vs cs] instance created looks like following. Although it uses the same repository as GitHub Codespaces uses, GitHub Codespaces has pushed nothing, [VS CS][vs cs] doesn't have the change.

![][image-07]

---

So far, we've walked through how to set up the dev environment in [VS CS][vs cs] for .NET Core application development. As this is the starting point of the team project efforts, it will significantly reduce the "It works on my machine" issue.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/setting-up-vs-codespaces-for-dotnet-core-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/setting-up-vs-codespaces-for-dotnet-core-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/setting-up-vs-codespaces-for-dotnet-core-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/setting-up-vs-codespaces-for-dotnet-core-04.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/setting-up-vs-codespaces-for-dotnet-core-05.png
[image-06]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/setting-up-vs-codespaces-for-dotnet-core-06.png
[image-07]: https://sa0blogs.blob.core.windows.net/devkimchi/2020/08/setting-up-vs-codespaces-for-dotnet-core-07.png

[devto post]: https://dev.to/n3wt0n/visual-studio-github-codespaces-questions-answered-5ge7

[vs cs]: https://visualstudio.microsoft.com/services/visual-studio-codespaces/?WT.mc_id=devkimchicom-blog-juyoo
[vs cs blog]: https://devblogs.microsoft.com/visualstudio/introducing-visual-studio-codespaces/?WT.mc_id=devkimchicom-blog-juyoo
[vs cs config]: https://docs.microsoft.com/visualstudio/codespaces/reference/configuring?WT.mc_id=devkimchicom-blog-juyoo
[vs cs personal]: https://docs.microsoft.com/visualstudio/codespaces/reference/personalizing?WT.mc_id=devkimchicom-blog-juyoo
[vs cs vso]: https://online.visualstudio.com/?WT.mc_id=devkimchicom-blog-juyoo

[gh cs]: https://github.com/features/codespaces/
[gh ohmyzsh]: https://github.com/ohmyzsh/ohmyzsh
[gh sample]: https://github.com/devkimchi/codespaces-dotnetcore
[gh vs cs config]: https://github.com/microsoft/vscode-dev-containers/tree/master/containers/dotnetcore

[dotnet core]: https://docs.microsoft.com/dotnet/?WT.mc_id=devkimchicom-blog-juyoo

[az]: https://azure.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo
[az free]: https://azure.microsoft.com/free/?WT.mc_id=devkimchicom-blog-juyoo
[az vso pricing]: https://azure.microsoft.com/pricing/details/visual-studio-online/?WT.mc_id=devkimchicom-blog-juyoo

[docker hub dotnet core]: https://hub.docker.com/_/microsoft-dotnet-core-sdk/
