---
title: "Generating Open API Document through Command-Line for Azure Functions"
slug: generating-open-api-doc-for-azure-functions-in-command-line
description: "This post shows how to generate an Open API document for Azure Functions, using a CLI."
date: "2020-07-08"
author: Justin-Yoo
tags:
- swagger
- azure-functions
- open-api
- cli
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2020/07/generating-open-api-doc-for-azure-functions-in-command-line-00.png
fullscreen: true
---

Almost 1.5 years ago, I introduced the [Swagger UI extension for Azure Functions][post prev] and got a lot of feedback. One of them is to implement a CLI that generates the Open API document for the Azure Functions app. As I recently released this CLI, it's a good time to introduce the tool through this post.


## Download CLI ##

[GitHub repository][gh release] has the CLI to download. As it's always tagged with `cli-<version>`, you can download the latest version of the CLI. In addition to this, the Open API extension supports [Azure Functions][az func] from v1 to the latest one. Therefore, depending on your needs, download the appropriate CLI, based on your Azure Functions app runtime and operating system.

* For Azure Functions v1
  * Windows only: `azfuncopenapi-v<version>-net461-win-x64.zip`
* For Azure Functions v2 or later
  * Linux: `azfuncopenapi-v<version>-netcoreapp3.1-linux-x64.zip`
  * MacOS: `azfuncopenapi-v<version>-netcoreapp3.1-osx-x64.zip`
  * Windows: `azfuncopenapi-v<version>-netcoreapp3.1-win-x64.zip`


## Generate Open API Document ##

Once you download the CLI above and implement the [Open API extension][gh doc openapi] on your Azure Functions app, you're good to go. Run the command below:

* Windows CLI:

https://gist.github.com/justinyoo/6da783b29c1f71fbee6c1f3b9bd59f6b?file=01-azfuncopenapi.ps1

* Linux/Mac CLI:

https://gist.github.com/justinyoo/6da783b29c1f71fbee6c1f3b9bd59f6b?file=02-azfuncopenapi.sh

Here are the options:

| Option | Description | Default Value |
| --- | --- | --- |
| `--project|-p` | Project path. It can be a fully qualified project path including `.csproj` or project directory. | Current directory |
| `--configuration|-c` | Configuration value. It can be either `Debug`, `Release` or something else. | `Debug` |
| `--target|-t` | Target framework. It should be `net4x` (eg. `net461`) for Azure Functions v1, `netcoreapp2.x` (eg. `netcoreapp2.1`) for Azure Functions v2, and `netcoreapp3.x` (eg. `netcoreapp3.1`) for Azure Functions v3. | `netcoreapp2.1` |
| `--version|-v` | Open API spec version. It should be either `v2` or `v3`. | `v2` |
| `--format|-f` | Open API document format. It should be either `json` or `yaml`. | `json` |
| `--output|-o` | Output directory for the generated Open API document. It can be a fully qualified directory path or relative path from `<PROJECT_ROOT>/bin/<CONFIGURATION>/<TARGET_FRAMEWORK>`. | `output` |
| `--console` | Value indicating whether to display the generated document to console or not. | `false` |

Run the CLI, and you'll be able to find either `swagger.json` or `swagger.yaml` generated in your designated output directory.

---

So far, I've introduced the CLI to generate Open API document for Azure Functions app. This tool is particularly useful when you work with [API Management][az apim] or [custom connectors][az cuscon] for [Power Platform][power platform] that require the Open API document.


[post prev]: /2019/02/02/introducing-swagger-ui-on-azure-functions/

[gh release]: https://github.com/aliencube/AzureFunctions.Extensions/releases
[gh doc openapi]: https://github.com/aliencube/AzureFunctions.Extensions/blob/dev/docs/openapi.md

[az func]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=devkimchicom-blog-juyoo
[az apim]: https://docs.microsoft.com/azure/api-management/api-management-key-concepts?WT.mc_id=devkimchicom-blog-juyoo
[az cuscon]: https://docs.microsoft.com/connectors/custom-connectors/?WT.mc_id=devkimchicom-blog-juyoo

[power platform]: https://powerplatform.microsoft.com/?WT.mc_id=devkimchicom-blog-juyoo
