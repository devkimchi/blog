---
title: "Deploying .NET Apps to Azure Container Apps with One Command, azd up"
slug: azdevfying-dotnet-apps-to-aca
description: "This post discusses how to provision and deploy .NET apps, including Azure Functions, using the single command azd up."
date: "2024-09-06"
author: Justin-Yoo
tags:
- dotnet
- containers
- azure-container-apps
- azd
cover: /2024/09/azdevfying-dotnet-apps-to-aca-00.png
fullscreen: true
---

In my previous blog posts of [containerising .NET apps][post prev 1] and [Function apps][post prev 2], I discussed how to containerise .NET apps and Azure Functions apps with and without `Dockerfile`. However, deploying these containerised apps to [Azure Container Apps (ACA)][az ca] is a different story. Since its release in May 2023, [Azure Developer CLI (`azd`)][azd] has evolved significantly. `azd` nowadays even automatically generates [Bicep][az bicep] files for us to immediately provision and deploy applications to Azure. With this feature, you only need the `azd up` command for provisioning and deployment. Throughout this post, I'm going to discuss how to provision and deploy .NET apps including Azure Functions to ACA through just one command, `azd up`.

> You can find a sample code from this [GitHub repository][gh sample].

## Prerequisites

There are a few prerequisites to containerise .NET apps effectively.

- [.NET SDK 8.0+][dotnet sdk]
- [Visual Studio][vs] or [Visual Studio Code][vs code] + [C# Dev Kit][vs code extension csharp]
- [Azure Developer CLI][azd]
- [Azure Functions Core Tools][az func cli]
- [Docker Desktop][docker desktop]

## Running the app locally

The [sample app repository][gh sample] already includes the following apps:

- [Blazor][aspnet blazor] app as a frontend
- [ASP.NET Core Web API][aspnet minimal api] app as a backend
- [Azure Functions][az func] app as another backend

Let's make sure those apps running properly on your local machine. In order to run those apps locally, open three terminal windows and run the following commands on each terminal:

```powershell
# Terminal 1 - ASP.NET Core Web API
dotnet run --project ./ApiApp

# Terminal 2 - Azure Functions
cd ./FuncApp
dotnet clean && func start

# Terminal 3 - Blazor app
dotnet run --project ./WebApp
```

Open your web browser and navigate to `https://localhost:5001` to see the Blazor app running. Then navigate to `https://localhost:5001/weather` to see the weather data fetched from the `ApiApp` and the greetings populated from the `FuncApp`.

![All apps up & running][image-01]

Now, let's start using `azd` to provision and deploy these apps to ACA. Make sure that you've already logged in to Azure with the `azd auth login` command.

## `azd init` &ndash; Initialisation

In order to provision and deploy the apps to ACA, you need to initialise the `azd` configuration. Run the following command:

```powershell
azd init
```

You'll be prompted to initialise the app. Choose the `Use code in the current directory` option.

![Use code in the current directory][image-02]

`azd` automatically detects your three apps as shown below. In addition to that, it says it will use Azure Container Apps. Choose the `Confirm and continue initializing my app` option.

![Confirm and continue][image-03]

The function app asks the target port number. Enter `80`.

![Enter the target port number][image-04]

And finally, it asks the environment name. Enter any name you want. I just entered `aca0906` for now.

![Enter the environment name][image-05]

Now, you've got two directories and two files generated:

- `.azure` directory
- `infra` directory
- `next-steps.md` file
- `azure.yaml` file

![Directories and files generated][image-06]

Under the `infra` directory, there are bunch of Bicep files automatically generated through `azd init`.

![Bicep files generated][image-07]

As a result of running the command, `azd init`, you don't have to write all necessary Bicep files. Instead, it generates them for you, which significantly reduces the time for infrastructure provisioning. Now, you're ready to provision and deploy your apps to ACA. Let's move on.

## `azd up` &ndash; Provision and deployment

All you need to run at this stage is:

```powershell
azd up
```

Then, it asks you to confirm the subscription and location to provision the resources. Choose the appropriate options and continue.

![Choose subscription and location for resource provisioning][image-08]

All apps are containerised and deployed to ACA. Once the deployment is done, you can see the output as shown below:

![Deployment done][image-09]

Click the web app URL and navigate to the `/weather` page. But you will see the error as shown below:

![Error on the web app][image-10]

This is because each app doesn't know where each other is. Therefore, you should update the Bicep files to let the web app know where the other apps are.

## Update Bicep files &ndash; Service discovery

Open the `infra/main.bicep` file and update the `webApp` resource:

```json
module webApp './app/WebApp.bicep' = {
  name: 'WebApp'
  params: {
    ...
    // Add these two lines
    apiAppEndpoint: apiApp.outputs.uri
    funcAppEndpoint: funcApp.outputs.uri
  }
  scope: rg
}
```

Then, open the `infra/app/WebApp.bicep` file and add both `apiAppEndpoint` and `funcAppEndpoint` parameters:

```json
...
@secure()
param appDefinition object

// Add these two lines
param apiAppEndpoint string
param funcAppEndpoint string
...
```

In the same file, change the `env` variable:

```json
// Before
var env = map(filter(appSettingsArray, i => i.?secret == null), i => {
  name: i.name
  value: i.value
})

// After
var env = union(map(filter(appSettingsArray, i => i.?secret == null), i => {
  name: i.name
  value: i.value
}), [
  {
    name: 'API_ENDPOINT_URL'
    value: apiAppEndpoint
  }
  {
    name: 'FUNC_ENDPOINT_URL'
    value: funcAppEndpoint
  }
])
```

This change passes the API and Function app endpoints to the web app as environment variables, so that the web app knows where the other apps are.

Once you've made the changes, run the `azd up` command again. It will update the resources in ACA. After that, go to the web app URL and navigate to the `/weather` page. You will see the weather data and greetings fetched from the API and Function apps.

![All apps up & running][image-11]

---

So far, I've discussed how to provision and deploy .NET apps including Azure Functions to ACA with just one command, `azd up`. This is a very convenient way to deploy apps to Azure. However, to let the apps know each other, you should slightly tweak the auto-generated Bicep files. With this little tweak, all your .NET apps will be seamlessly provisioned and deployed to ACA.

One more thing I'd like to mention here, though, is that, if you use [.NET Aspire][dotnet aspire], this sort of service discovery is automatically handled.

## More about deploying .NET apps to ACA?

If you want to learn more options about deploying .NET apps to ACA, the following links might be helpful.

- [Deployment options to ACA][az ca deployment options]
- [Publish .NET apps to ACA via Azure CLI][az ca deploy via az cli]
- [.NET Aspire][dotnet aspire]

[image-01]: /2024/09/azdevfying-dotnet-apps-to-aca-01.png
[image-02]: /2024/09/azdevfying-dotnet-apps-to-aca-02.png
[image-03]: /2024/09/azdevfying-dotnet-apps-to-aca-03.png
[image-04]: /2024/09/azdevfying-dotnet-apps-to-aca-04.png
[image-05]: /2024/09/azdevfying-dotnet-apps-to-aca-05.png
[image-06]: /2024/09/azdevfying-dotnet-apps-to-aca-06.png
[image-07]: /2024/09/azdevfying-dotnet-apps-to-aca-07.png
[image-08]: /2024/09/azdevfying-dotnet-apps-to-aca-08.png
[image-09]: /2024/09/azdevfying-dotnet-apps-to-aca-09.png
[image-10]: /2024/09/azdevfying-dotnet-apps-to-aca-10.png
[image-11]: /2024/09/azdevfying-dotnet-apps-to-aca-11.png

[gh sample]: https://github.com/devkimchi/azdevfying-dotnet-apps-on-aca

[post prev 1]: /2024/08/12/different-containerising-options-for-dotnet-devs/
[post prev 2]: /2024/08/23/containerising-azure-functions-without-dockerfile/

[aspnet blazor]: https://learn.microsoft.com/aspnet/core/blazor/?WT.mc_id=dotnet-149673-juyoo
[aspnet minimal api]: https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview?WT.mc_id=dotnet-149673-juyoo

[az bicep]: https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview?tabs=bicep?WT.mc_id=dotnet-149673-juyoo

[az ca]: https://learn.microsoft.com/azure/container-apps/overview?WT.mc_id=dotnet-149673-juyoo
[az ca deployment options]: https://learn.microsoft.com/azure/container-apps/code-to-cloud-options?WT.mc_id=dotnet-149673-juyoo
[az ca deploy via az cli]: https://learn.microsoft.com/azure/container-apps/quickstart-code-to-cloud?tabs=bash%2Ccsharp&WT.mc_id=dotnet-149673-juyoo

[azd]: https://learn.microsoft.com/azure/developer/azure-developer-cli/overview?WT.mc_id=dotnet-149673-juyoo

[az func]: https://learn.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-149673-juyoo
[az func cli]: https://learn.microsoft.com/azure/azure-functions/functions-run-local?WT.mc_id=dotnet-149673-juyoo

[dotnet sdk]: https://dotnet.microsoft.com/download/dotnet/8.0?WT.mc_id=dotnet-149673-juyoo
[dotnet aspire]: https://learn.microsoft.com/dotnet/aspire/get-started/aspire-overview?WT.mc_id=dotnet-149673-juyoo

[docker desktop]: https://docs.docker.com/desktop/

[vs]: https://visualstudio.microsoft.com/?WT.mc_id=dotnet-149673-juyoo
[vs code]: https://code.visualstudio.com/?WT.mc_id=dotnet-149673-juyoo
[vs code extension csharp]: https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit?WT.mc_id=dotnet-149673-juyoo
