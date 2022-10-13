---
title: "DevContainers for .NET developers on Azure"
slug: devcontainers-for-dotnet-devs-on-azure
description: "In this post, I'm going to discuss what the devcontainer is and what configurations are useful for .NET developers on Azure."
date: "2022-10-12"
author: Justin-Yoo
tags:
- azure
- dotnet
- devcontainers
- gh-codespaces
cover: /2022/10/devcontainers-for-dotnet-devs-on-azure-00.png
fullscreen: true
---

Containers are everywhere. It's not just to make both the development and production environment consistent but also to build CI/CD pipelines, test automations, and so on. While everyone does their own way of building containers, it has become the maintenance job.

Maintenance will become easier if it uses standardised ways. What if there is an open standard spec for it? Actually, there is. It is called [Development Container or DevContainer][container dev]. You can build the container and store it in your GitHub repository with this spec. Then, [GitHub Codespaces][gh codespaces] makes use of it. At least a project using the same GitHub repository uses the same development environment. Throughout this post, I'm going to introduce the DevContainer spec and discuss how it's used for Azure and .NET app development.

> You can download the DevContainer template from this [GitHub repository][gh sample], which is used for this post.


## Elements of DevContainer ##

All you need for building a DevContainer are those two files:

* `devcontainer.json`: It defines all the metadata details to build a DevContainer.
* `Dockerfile`: It defines the Docker container image that is invoked within `devcontainer.json`.

Optionally, you can call a shell script during the container building lifecycle. The `postCreateCommand` attribute of the `devcontainer.json` file takes care of it. For example, the attribute executes the `post-create.sh` file.

> Alternatively, you can invoke the `RUN` command within `Dockerfile` for the shell script execution. In this case, you're running the script with the `root` permissions. On the other hand, `postCreateCommand` runs the script with the user privileges declared in `devcontainer.json`.


## Structure of `devcontainer.json` ##

`devcontainer.json` consists of various sections for metadata declarations. I only deal with a small portion of it in this post.

* `build`: This section is for the Docker container build &ndash; location of `Dockerfile`, parameters passed to it.
* `forwardPorts`: This section defines the list of port numbers to expose. For example, `7071` is for Azure Functions, `5000` and `5001` are for ASP.NET Web/API apps, and `4280` is for Azure Static Web App development.
* `features`: While you can add everything in `Dockerfile` for the build, there are already pre-configured [features][container dev features] you can optionally add. You can find the complete list of the features at [here][container dev features list]. Some examples of those features are common utilities and tools like Azure CLI, GitHub CLI and Terraform, and languages like node.js, Java, .NET, Python, etc.
* `customisations`: This section is responsible for the tools used in DevContainer. One of the tools is VS Code which the `vscode` attribute represents. It defines which extensions need to be included (`extensions`) and how the editor behaves (`settings`).
* `remoteUser`: This sets the user account within DevContainer. Unless it is set, DevContainer runs as the `root` account. In this post, we use `vscode` as the non-root account.
* `postCreateCommand`: Once DevContainer is created, run additional commands with the remote-user account privileges. Through this attribute, you can run an additional shell script.

> There are more attributes than the ones mentioned above. If you want to know more, visit [this page][container dev spec].


## Order of Building DevContainer ##

How does DevContainer get built? Here is the rough order.

1. Build the Docker container. If you add the shell script through the `RUN` command in `Dockerfile`, the shell script is run this time.
2. Run features declared in the `features` section of `devcontainer.json` while building the Docker container.
3. Run commands declared in the `postCreateCommand` attribute of `devcontainer.json`.
4. Apply [dotfiles][container dev dotfiles] after `postCreateCommand`, if you have it.
5. Apply both `extensions` and `settings` of `devcontainer.json` at the startup of the DevContainer.

By understanding the order above, let's build the DevContainer for .NET app development on Azure.


## Base Container Image ##

When you visit the [repository for DevContainer images][container dev images], there are pre-configured [list of base Docker container images for DevContainer][container dev images list]. Let's choose the one for .NET. You can choose many different variants, but either `6.0-jammy` (Ubuntu 22.04) or `6.0-focal` (Ubuntu 20.04) would be yours if you prefer using Ubuntu-based images with .NET 6. Ubuntu 22.04 is set to the default base image in this post.

```dockerfile
# [Choice] .NET version: 6.0-jammy, 6.0-focal
ARG VARIANT="6.0-jammy"
FROM mcr.microsoft.com/dotnet/sdk:${VARIANT}
```


## `devcontainer.json` Configuration ##

Let's configure `devcontainer.json` containing all the metadata details.


### The `build` Section ###

It declares the location of `Dockerfile` and sends parameters to it. In this case, use `6.0-jammy` for `VARIANT`.

```jsonc
"build": {
  "dockerfile": "./Dockerfile",
  "context": ".",
  "args": {
    "VARIANT": "6.0-jammy"
    // Use this only if you need Razor support, until OmniSharp supports .NET 6 properly
    // "VARIANT": "6.0-focal"
  }
}
```

> If you are building a Blazor app, pass `6.0-focal` (Ubuntu 20.04) instead of `6.0-jammy` (Ubuntu 22.04). It's because the [C# extension][vscode extensions csharp] has a [bug][vscode extensions csharp bug] on Ubuntu 22.04.


### The `forwardPorts` Section ###

You might need to expose some specific port numbers &ndash; `7071` for Azure Functions, `5000` and `5001` for ASP.NET Web/API apps, and/or `4280` for Azure Static Web Apps.

```jsonc
"forwardPorts": [
  // Azure Functions
  7071,
  // ASP.NET Core Web/API App, Blazor App
  5000, 5001,
  // Azure Static Web App CLI
  4280
]
```


### The `features` Section ###

On top of the base image, if you want to add more tools and/or languages, add them to the `features` section.

* **Common Utils**: You can add zsh and oh-my-zsh through this feature.

    ```jsonc
    "features": {
      // Install common utilities
      "ghcr.io/devcontainers/features/common-utils:1": {
        "installZsh": true,
        "installOhMyZsh": true,
        "upgradePackages": true,
        "username": "vscode",
        "uid": "1000",
        "gid": "1000"
      }
    }
    ```

* **Azure CLI**: You can add Azure CLI through this feature.

    ```jsonc
    "features": {
      // Uncomment the below to install Azure CLI
      "ghcr.io/devcontainers/features/azure-cli:1": {
        "version": "latest"
      }
    }
    ```

* **GitHub CLI**: You can add GitHub CLI through this feature.

    ```jsonc
    "features": {
      // Uncomment the below to install GitHub CLI
      "ghcr.io/devcontainers/features/github-cli:1": {
        "version": "latest"
      }
    }
    ```

* **node.js**: You can add the latest LTS version of node.js through this feature.

    ```jsonc
    "features": {
      // Uncomment the below to install node.js
      "ghcr.io/devcontainers/features/node:1": {
        "version": "lts",
        "nodeGypDependencies": true,
        "nvmInstallPath": "/usr/local/share/nvm"
      }
    }
    ```

Suppose you have another feature, but it doesn't exist yet in [this features list][container dev features list]. In that case, you can manually add it through the `postCreateCommand` attribute by invoking `post-create.sh`.

Do you want to add those features in order? Then use this `overrideFeatureInstallOrder` attribute. Here in this post, the `common-utils` feature runs first, and then the rest features are installed in random order.

```jsonc
"overrideFeatureInstallOrder": [
  "ghcr.io/devcontainers/features/common-utils"
]
```


### The `customizations.vscode.extensions` Section ###

You might have some extensions automatically installed while creating the DevContainer. The `customisations.vscode.extensions` section holds all the extensions you want to install. The list of extensions below is for Azure and .NET app development. If you want to add more, search them on [Visual Studio Code Marketplace][vscode marketplace] and add their extension ID. For example, the [C# extension][vscode extensions csharp] has its extension ID of `ms-dotnettools.csharp`.

```jsonc
"customizations": {
  "vscode": {
    "extensions": [
      // Recommended extensions - GitHub
      "cschleiden.vscode-github-actions",
      "GitHub.vscode-pull-request-github",

      // Recommended extensions - Azure
      "ms-azuretools.vscode-bicep",

      // Recommended extensions - Collaboration
      "eamodio.gitlens",
      "EditorConfig.EditorConfig",
      "MS-vsliveshare.vsliveshare-pack",
      "streetsidesoftware.code-spell-checker",

      // Recommended extensions - .NET
      "Fudge.auto-using",
      "jongrant.csharpsortusings",
      "kreativ-software.csharpextensions",

      // Recommended extensions - Power Platform
      "microsoft-IsvExpTools.powerplatform-vscode",

      // Recommended extensions - Markdown
      "bierner.github-markdown-preview",
      "DavidAnson.vscode-markdownlint",
      "docsmsft.docs-linting",
      "johnpapa.read-time",
      "yzhang.markdown-all-in-one",

      // Required extensions
      "GitHub.copilot",
      "ms-dotnettools.csharp",
      "ms-vscode.PowerShell",
      "ms-vscode.vscode-node-azure-pack",
      "VisualStudioExptTeam.vscodeintellicode"
    ]
  }
}
```


### The `customizations.vscode.settings` Section ###

You might also want to personalise your editor settings while creating the DevContainer. You can set them on the `customizations.vscode.settings` section. The list of settings below are examples of personalised settings. If you want to get them more personalised, refer to the [User and Workspace Settings][vscode settings] page.

* The `bash` shell is the default terminal. If you want to change its default behaviour to `zsh`, use these settings.

    ```jsonc
    "customizations": {
      "vscode": {
        "settings": {
          // Uncomment if you want to use zsh as the default shell
          "terminal.integrated.defaultProfile.linux": "zsh",
          "terminal.integrated.profiles.linux": {
            "zsh": {
              "path": "/usr/bin/zsh"
            }
          }
        }
      }
    }
    ```

* If you want to change your terminal font, use this setting. It's specifically for oh-my-zsh or oh-my-posh.

    ```jsonc
    "customizations": {
      "vscode": {
        "settings": {
          // Uncomment if you want to use CaskaydiaCove Nerd Font as the default terminal font
          "terminal.integrated.fontFamily": "CaskaydiaCove Nerd Font"
        }
      }
    }
    ```

* If you want to disable the minimap feature, use this setting.

    ```jsonc
    "customizations": {
      "vscode": {
        "settings": {
          // Uncomment if you want to disable the minimap view
          "editor.minimap.enabled": false
        }
      }
    }
    ```

* If you want to change the behaviour of the explorer, use these settings. All files are sorted by extension and nested by relevant files.

    ```jsonc
    "customizations": {
      "vscode": {
        "settings": {
          // Recommended settings for the explorer pane
          "explorer.sortOrder": "type",
          "explorer.fileNesting.enabled": true,
          "explorer.fileNesting.patterns": {
            "*.bicep": "${capture}.json",
            "*.razor": "${capture}.razor.css",
            "*.js": "${capture}.js.map"
          }
        }
      }
    }
    ```


### The `postCreateCommand` Section ###

Once your DevContainer is created, you might want to do something more. For example, you can't add an extra feature through the `features` section because it's not ready. However, executing a shell script can add those additional features through this `postCreateCommand` section.

Here's the one for running `post-create.sh` through the `bash` shell.

```jsonc
"postCreateCommand": "/bin/bash ./.devcontainer/post-create.sh > ~/post-create.log",
```

Here's the one for running `post-create.sh` through the `zsh` shell.

```jsonc
"postCreateCommand": "/usr/bin/zsh ./.devcontainer/post-create.sh > ~/post-create.log",
```

Let's take a look at what happens within `post-create.sh`.


## `post-create.sh` Execution ##

This `post-create.sh` is run right after the DevContainer is created.


### CaskaydiaCove Nerd Font ###

It's a good idea to install a custom font, if you use either oh-my-zsh or oh-my-posh for your terminal. The following script is to install CaskaydiaCove Nerd Font.

```powershell
## CaskaydiaCove Nerd Font
# Uncomment the below to install the CaskaydiaCove Nerd Font
mkdir $HOME/.local
mkdir $HOME/.local/share
mkdir $HOME/.local/share/fonts
wget https://github.com/ryanoasis/nerd-fonts/releases/latest/download/CascadiaCode.zip
unzip CascadiaCode.zip -d $HOME/.local/share/fonts
rm CascadiaCode.zip
```


### Azure CLI Extensions ###

You can run this script if you install Azure CLI through the `feature` section of `devcontainer.json`. However, because it adds ALL extensions, it takes about 30-60 mins, depending on your network latency. Therefore be careful to use.

```powershell
## AZURE CLI EXTENSIONS ##
# Uncomment the below to install Azure CLI extensions
extensions=$(az extension list-available --query "[].name" | jq -c -r '.[]')
for extension in $extensions;
do
    az extension add --name $extension
done
```

> You can use `extensions=(list of selected extensions you want)`, instead of using `extensions=$(az extension list-available --query "[].name" | jq -c -r '.[]')`. For example, I use `extensions=(account alias deploy-to-azure functionapp subscription webapp)`.


### Azure Bicep CLI ###

If you use Azure Bicep, you can run this script to install Bicep CLI.

```powershell
## AZURE BICEP CLI ##
# Uncomment the below to install Azure Bicep CLI
az bicep install
```


### Azure Functions Core Tools ###

Do you develop Azure Functions apps? Then activate this script. As it installs through npm, you should install the node.js feature through the `features` section of `devcontainer.json`.

```powershell
## AZURE FUNCTIONS CORE TOOLS ##
# Uncomment the below to install Azure Functions Core Tools
npm i -g azure-functions-core-tools@4 --unsafe-perm true
```


### Azure Static Web Apps CLI ###

Unlock this script if you're building a Blazor WASM app and deploying it to Azure Static Web Apps. Like Azure Functions Core Tools, it relies on the node.js feature through the `features` section of `devcontainer.json`.

```powershell
## AZURE STATIC WEB APPS CLI ##
# Uncomment the below to install Azure Static Web Apps CLI
npm install -g @azure/static-web-apps-cli
```


### Azure Dev CLI ###

If you want to activate the Azure Dev CLI, run the following script. Azure Dev CLI needs both Azure CLI and GitHub CLI as dependencies. Therefore, make sure you have already installed them through the `features` section of `devcontainer.json`.

```powershell
## AZURE DEV CLI ##
# Uncomment the below to install Azure Dev CLI
curl -fsSL https://aka.ms/install-azd.sh | bash
```


### oh-my-zsh &ndash; Plugins and Themes ###

There are a bunch of plugins and themes for oh-my-zsh. You can follow the pattern below. Here are some recommended plugins and theme &ndash; powerlevel10k.

```powershell
## OH-MY-ZSH PLUGINS & THEMES (POWERLEVEL10K) ##
# Uncomment the below to install oh-my-zsh plugins and themes (powerlevel10k) without dotfiles integration
git clone https://github.com/zsh-users/zsh-completions.git $HOME/.oh-my-zsh/custom/plugins/zsh-completions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git $HOME/.oh-my-zsh/custom/plugins/zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-autosuggestions.git $HOME/.oh-my-zsh/custom/plugins/zsh-autosuggestions

git clone https://github.com/romkatv/powerlevel10k.git $HOME/.oh-my-zsh/custom/themes/powerlevel10k --depth=1
ln -s $HOME/.oh-my-zsh/custom/themes/powerlevel10k/powerlevel10k.zsh-theme $HOME/.oh-my-zsh/custom/themes/powerlevel10k.zsh-theme
```


### oh-my-zsh &ndash; powerlevel10k Theme Configurations ###

powerlevel10k has its settings file, called `p10k.zsh`. I've got my own `p10k.zsh` settings &ndash; with a clock and without the clock. Activate the script below to copy them to DevContainer. You don't need to run the following script if you have your own ones from your `dotfiles` repository.

```powershell
## OH-MY-ZSH - POWERLEVEL10K SETTINGS ##
# Uncomment the below to update the oh-my-zsh settings without dotfiles integration
curl https://raw.githubusercontent.com/justinyoo/devcontainers-dotnet/main/oh-my-zsh/.p10k-with-clock.zsh > $HOME/.p10k-with-clock.zsh
curl https://raw.githubusercontent.com/justinyoo/devcontainers-dotnet/main/oh-my-zsh/.p10k-without-clock.zsh > $HOME/.p10k-without-clock.zsh
curl https://raw.githubusercontent.com/justinyoo/devcontainers-dotnet/main/oh-my-zsh/switch-p10k-clock.sh > $HOME/switch-p10k-clock.sh
chmod +x ~/switch-p10k-clock.sh

cp $HOME/.p10k-with-clock.zsh $HOME/.p10k.zsh
cp $HOME/.zshrc $HOME/.zshrc.bak

echo "$(cat $HOME/.zshrc)" | awk '{gsub(/ZSH_THEME=\"codespaces\"/, "ZSH_THEME=\"powerlevel10k\"")}1' > $HOME/.zshrc.replaced && mv $HOME/.zshrc.replaced $HOME/.zshrc
echo "$(cat $HOME/.zshrc)" | awk '{gsub(/plugins=\(git\)/, "plugins=(git zsh-completions zsh-syntax-highlighting zsh-autosuggestions)")}1' > $HOME/.zshrc.replaced && mv $HOME/.zshrc.replaced $HOME/.zshrc
echo "
# To customize prompt, run 'p10k configure' or edit ~/.p10k.zsh.
[[ ! -f ~/.p10k.zsh ]] || source ~/.p10k.zsh
" >> $HOME/.zshrc
```


### oh-my-posh &ndash; Installation ###

Are you into PowerShell but also want to use something like oh-my-zsh? Then oh-my-posh is your friend. Run the following script to install.

```powershell
## OH-MY-POSH ##
# Uncomment the below to install oh-my-posh
sudo wget https://github.com/JanDeDobbeleer/oh-my-posh/releases/latest/download/posh-linux-amd64 -O /usr/local/bin/oh-my-posh
sudo chmod +x /usr/local/bin/oh-my-posh
```


### oh-my-posh &ndash; Configurations ###

oh-my-posh has its own powerlevel10k configurations. Run the following script to copy those settings files to your DevContainer. Again, if you've set them on your `dotfiles` repository, you don't need to run the script below.

```powershell
## OH-MY-POSH - POWERLEVEL10K SETTINGS ##
# Uncomment the below to update the oh-my-posh settings without dotfiles integration
curl https://raw.githubusercontent.com/justinyoo/devcontainers-dotnet/main/oh-my-posh/p10k-with-clock.omp.json > $HOME/p10k-with-clock.omp.json
curl https://raw.githubusercontent.com/justinyoo/devcontainers-dotnet/main/oh-my-posh/p10k-without-clock.omp.json > $HOME/p10k-without-clock.omp.json
curl https://raw.githubusercontent.com/justinyoo/devcontainers-dotnet/main/oh-my-posh/switch-p10k-clock.ps1 > $HOME/switch-p10k-clock.ps1

mkdir $HOME/.config/powershell
curl https://raw.githubusercontent.com/justinyoo/devcontainers-dotnet/main/oh-my-posh/Microsoft.PowerShell_profile.ps1 > $HOME/.config/powershell/Microsoft.PowerShell_profile.ps1

cp $HOME/p10k-with-clock.omp.json $HOME/p10k.omp.json
```

Once you complete the configuration like above, run your GitHub Codespaces instance. Then, you will see the terminal like below. On the left-hand side, it's the zsh shell applying oh-my-zsh. On the right-hand side, it's PowerShell using oh-my-posh.

![GH Codespaces][image-01]

---

So far, we've discussed what the [DevContainer][container dev] is, how it is helpful for .NET app development on Azure, and what needs to be configured for DevContainer for .NET app development on Azure, using [GitHub Codespaces][gh codespaces]. What has been discussed here in this post is the only small part of the DevContainer. Therefore, if you want to apply it to your or your team's development environment, you need to do more research by yourself. However, I hope this post gives at least some sort of insights for building DevContainers.


## Want to know more about DevContainers? ##

There are documents and learning materials for you:

* [Beginner's Series to: Dev Containers][container dev learn]
* [Create a development container][container dev create]
* [GitHub Codespaces overview][gh codespaces]
* [Add a dev container configuration to your repository][container dev codespaces]


[image-01]: /2022/10/devcontainers-for-dotnet-devs-on-azure-01.png


[gh sample]: https://github.com/justinyoo/devcontainers-dotnet
[gh codespaces]: https://docs.github.com/codespaces/overview

[container dev]: https://containers.dev/
[container dev spec]: https://github.com/devcontainers/spec
[container dev images]: https://github.com/devcontainers/images
[container dev images list]: https://github.com/devcontainers/images/tree/main/src
[container dev features]: https://github.com/devcontainers/features
[container dev features list]: https://github.com/devcontainers/features/tree/main/src

[container dev create]: https://code.visualstudio.com/docs/remote/create-dev-container?WT.mc_id=dotnet-78728-juyoo
[container dev learn]: https://learn.microsoft.com/shows/beginners-series-to-dev-containers/?WT.mc_id=dotnet-78728-juyoo
[container dev codespaces]: https://docs.github.com/codespaces/setting-up-your-project-for-codespaces/setting-up-your-project-for-codespaces?langId=dotnet
[container dev dotfiles]: https://docs.github.com/codespaces/customizing-your-codespace/personalizing-github-codespaces-for-your-account

[vscode marketplace]: https://marketplace.visualstudio.com/VSCode?WT.mc_id=dotnet-78728-juyoo
[vscode settings]: https://code.visualstudio.com/docs/getstarted/settings?WT.mc_id=dotnet-78728-juyoo
[vscode extensions csharp]: https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp&WT.mc_id=dotnet-78728-juyoo
[vscode extensions csharp bug]: https://github.com/OmniSharp/omnisharp-vscode/issues/5347
