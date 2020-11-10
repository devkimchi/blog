---
title: "OpenAPI Extension to Support Azure Functions V1"
slug: openapi-extension-to-support-azure-functions-v1
description: "This post shows how Azure Functions v3 runtime works as a proxy to Azure Functions v1 runtime, to enable the Open API extension."
date: "2020-11-11"
author: Justin-Yoo
tags:
- azure-functions
- azure-functions-proxy
- openapi-extension
- backward-compatibility
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2020/11/openapi-extension-to-support-azure-functions-v1-00.png
fullscreen: true
---

There is an [open-source project][gh azfunc swagger] that [generates an Open API document on-the-fly][post azfunc swagger] on an [Azure Functions][az func] app. The open-source project also provides a [NuGet package library][nuget azfunc swagger]. This project has been supporting the whole Azure Functions runtimes from v1 since Day 1. Although it does, the v1 runtime has got its own limitation that cannot generate the Open API document automatically. But, as always, there's a workaround. I'm going to show how to generate the Open API document on-the-fly and execute the v1 function through the [Azure Function Proxy][az func proxy] feature.


## Legacy V1 Azure Function ##

Generally speaking, many legacy enterprise applications still need Azure Functions v1 runtime due to their reference dependencies. Let's assume that the Azure Functions v1 endpoint looks like the following:

https://gist.github.com/justinyoo/2b0b286bbe3e727e17423047cd97f86e?file=01-v1-legacy.cs

The v1 runtime has a strong tie to the [Newtonsoft.Json][nuget jsonnet] package version 9.0.1. Therefore, if the return object of `MyReturnObject` has a dependency on Newtonsoft.Json v10.0.1 and later, the [Open API extension][gh azfunc swagger] cannot be used.


## Azure Functions Proxy for Open API Document ##

The [Azure Functions Proxy][az func proxy] feature comes the rescue! Although it's not a perfect solution, it provides with the same developer experience, which is worth trying. Let's build an Azure Functions app targeting the v3 runtime. The name of the proxy function is `MyV1ProxyFunctionApp` (line #1). All the rest are set to be the same as the legacy v1 app (line #3-7). However, make sure this is the proxy purpose, meaning it does nothing but returns an OK response (line #10).

https://gist.github.com/justinyoo/2b0b286bbe3e727e17423047cd97f86e?file=02-v1-proxy.cs&highlights=1,3-7,10

Once installed the [Open API library][nuget azfunc swagger], let's add decorators above the `FunctionName(...)` decorator (line #5-9).

https://gist.github.com/justinyoo/2b0b286bbe3e727e17423047cd97f86e?file=03-v1-proxy-openapi.cs&highlights=5-9

All done! Run this proxy app, and you will be able to see the Swagger UI page. As I mentioned above, this app doesn't work but show the UI page. For this app to work, extra work needs to be done.


## proxies.json to Legacy Azure Functions V1 ##

Add the `proxies.json` file to the root folder. As we added the same endpoint as the legacy function app on purpose (line #6,11), API consumers should have the same developer experience as before except the hostname change. In addition to that, both querystring values and request headers are relayed to the legacy app (line #13-14).

https://gist.github.com/justinyoo/2b0b286bbe3e727e17423047cd97f86e?file=04-proxies.json&highlights=6,11,13-14

Then update the `.csproj` file to deploy the `proxies.json` file together (line #10-12).

https://gist.github.com/justinyoo/2b0b286bbe3e727e17423047cd97f86e?file=05-v1-proxy.csproj&highlights=10-12

All done! Run this proxy function on your local machine or deploy it to Azure, and hit the proxy API endpoint. Then you'll be able to see the Open API document generated on-the-fly and execute the legacy API through the proxy.

---

So far, we have created an [Azure Functions][az func] app using the [Azure Functions Proxy][az func proxy] feature. It also supports the [Open API document generation][gh azfunc swagger] for the v1 runtime app. The flip-side of this approach costs doubled because all API requests hit the proxy then the legacy. The cost optimisation should be investigated from the enterprise architecture perspective.


[post azfunc swagger]: /2019/02/02/introducing-swagger-ui-on-azure-functions/

[gh azfunc swagger]: https://github.com/aliencube/AzureFunctions.Extensions

[nuget azfunc swagger]: https://www.nuget.org/packages/Aliencube.AzureFunctions.Extensions.OpenApi/
[nuget jsonnet]: https://www.nuget.org/packages/Newtonsoft.Json/

[az func]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-10867-juyoo
[az func proxy]: https://docs.microsoft.com/azure/azure-functions/functions-proxies?WT.mc_id=dotnet-10867-juyoo
