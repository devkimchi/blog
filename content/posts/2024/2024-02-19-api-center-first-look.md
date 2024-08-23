---
title: "Azure API Center: The First Look"
slug: api-center-first-look
description: "Throughout this post, I'm going to take a first look what Azure API Center is and what I can do with it."
date: "2024-02-19"
author: Justin-Yoo
tags:
- azure
- api-center
- api-management
cover: /2024/02/api-center-first-look-00.png
fullscreen: true
---

Let's say you work for a company that has a lot of APIs. You might have a few questions:

- How do you manage the lifecycle of those APIs?
- What governance would you apply to these APIs?
- What is the list of environments needed to manage these APIs?
- What are the deployment strategies for those APIs?
- How would you integrate those APIs with other services?

As your company's number of APIs increases, so does the complexity of managing them. [Azure API Center (APIC)][apic] is a central repository for your APIs lifecycle management, and it offers the more efficient ways for management. Throughout this post, I will take a first look at what Azure API Center is and what it offers.

> You can find a sample code from this [GitHub repository][gh sample].

## Prerequisites

There are a few prerequisites to use Azure APIC effectively:

- [Visual Studio Code][vs code] with the [Azure API Center extension][vs code extension apic], [Rest Client extension][vs code extension rest] and [Kiota extension][vs code extension kiota]
- [Azure CLI][az cli] with the [API Center extension][az cli extension apic]
- [Azure Developer CLI][azd cli] for easy resource provisioning through [Bicep][az bicep]

## API Center instance provisioning

There are three ways to provision an APIC instance:

- Through Bicep
- Through Azure CLI
- Through [Azure Portal][apic portal]

I'm not going to discuss how to provision an APIC instance in this article. But here's the reference you can do it by yourself through Bicep &ndash; [Azure API Center Sample](https://github.com/devkimchi/api-center-sample)

## Register APIs to APIC

The purpose of using APIC is to manage your company's APIs in a centralised manner. From design to deployment, APIC tracks all the histories. To register your APIs to APIC, you can use either Azure CLI or Azure Portal.

Let's say there's a weather forecast API you have designed and developed. You have an OpenAPI document for the API, but not implemented yet. Let's register the API to APIC.

```powershell
az apic api register \
    -g "my-resource-group" \
    -s "my-api-center" \
    --api-location ./weather-forecast.json
```

![Registering API through Azure CLI][image-01]

If you want to register another API through the Azure Portal, you can do it by following the [official documentation][apic portal register].

![Registering API through Azure Portal][image-02]

## Import APIs from API Management to APIC

If you have already working APIs in [Azure API Management (APIM)][apim], you can import them to APIC through Azure CLI. But it requires a few more steps to do so.

1. First of all, you need to activate Managed Identity to the APIC instance. It can be either system identity or user identity, but I'm going to use the system identity for now.

    ```powershell
    az apic service update \
        -g "my-resource-group" \
        -s "my-api-center" \
        --identity '{ "type": "SystemAssigned" }'
    ```

1. Then, get the principal ID of the APIC instance.

    ```powershell
    APIC_PRINCIPAL_ID=$(az apic service show \
        -g "my-resource-group" \
        -s "my-api-center" \
        --query "identity.principalId" -o tsv)
    ```

1. Now, register the APIC instance to the APIM instance as an APIM reader.

    ```powershell
    APIM_RESOURCE_ID=$(az apim show \
        -g "my-resource-group" \
        -s "my-api-center" \
        --query "id" -o tsv)
    
    az role assignment create \
        --role "API Management Service Reader Role" \
        --assignee-object-id $APIC_PRINCIPAL_ID \
        --assignee-principal-type ServicePrincipal \
        --scope $APIM_RESOURCE_ID
    ```

1. And finally, import APIs from APIM to APIC.

    ```powershell
    az apic service import-from-apim \
        -g "my-resource-group" \
        -s "my-api-center" \
        --source-resource-ids "$APIM_RESOURCE_ID/apis/*"
    ```

   ![Importing API from APIM][image-03]

Now, you have registered and imported APIs to APIC. But registering those APIs to APIC does nothing to do with us. What's next then? Let's play around those APIs on Visual Studio Code.

## View APIs on Visual Studio Code &ndash; Swagger UI

So, what can you do with the APIs registered and imported to APIC? You can view the list of APIs on Visual Studio Code. First, you need to install the [Azure API Center extension][vs code extension apic] on Visual Studio Code.

Once you install the extension, you can see the list of APIs on the extension. Choose one of the APIs and right-click on it. Then, you can see the context menu. Click on the **Open API Documentation** menu item.

![Open API documentation][image-04]

You will see the Swagger UI page, showing your API document. With this Swagger UI, you can test your API endpoints.

![Swagger UI][image-05]

## Test APIs on Visual Studio Code &ndash; Rest Client

Although you can test your API endpoints on the Swagger UI, you can also test them in a different way. For this, you need to install the [Rest Client extension][vs code extension rest] on Visual Studio Code.

After you install the extension, choose one of the APIs and right-click on it. Then, you can see the context menu. Click on the **Generate HTTP File** menu item.

![Generate HTTP file][image-06]

Within the HTTP file, you can actually test your API endpoints with different payloads.

![HTTP file][image-07]

## Generate client SDK on Visual Studio Code &ndash; Kiota

You can write up the client SDK by yourself. But it's time consuming and fragile because the API can change at any time. But what if somebody or a tool creates the client SDK on your behalf?

One of the greatest features of this APIC extension offers is to generate client SDKs. You can generate the client SDKs for your APIs in different languages. Although the API itself has no implementation yet, you can still work with the client SDK because you know what you need to send and what you will receive in return through the SDK. For this, you need to install the [Kiota extension][vs code extension kiota] on Visual Studio Code.

After you install the extension, choose one of the APIs and right-click on it. Then, you can see the context menu. Click on the **Generate API Client** menu item.

![Generate API client][image-08]

Because I have a Blazor web application, I'm going to generate a C# client SDK for the API. The Kiota extension finds out all the API endpoints from APIC. You can choose them all or just a few of them. Click the ▶️ button, and it generates the client SDK for you.

![Kiota Explorer][image-09]

Add the necessary information like class name and namespace of the client SDK, and output folder. Finally it asks in which language to generate the client SDK. There are currently 9 languages available for now. I'm going to choose C#.

![Kiota - choose language][image-10]

The Kiota extension then generates the client SDK into the designated directory.

![API client SDK generated][image-11]

## Consume the generated client SDK within an application

Now, the client SDK has been generated by the Kiota extension from APIC to my Blazor application. Because it uses the Kiota libraries, I need to install the following Kiota NuGet packages to my Blazor web application.

```powershell
dotnet add ./src/WebApp/ package Microsoft.Kiota.Http.HttpClientLibrary
dotnet add ./src/WebApp/ package Microsoft.Kiota.Serialization.Form
dotnet add ./src/WebApp/ package Microsoft.Kiota.Serialization.Json
dotnet add ./src/WebApp/ package Microsoft.Kiota.Serialization.Text
dotnet add ./src/WebApp/ package Microsoft.Kiota.Serialization.Multipart
```

Add dependencies to the [`Program.cs` file](https://github.com/devkimchi/api-center-sample/blob/main/src/WebApp/Program.cs) and update the [`Home.razor` file](https://github.com/devkimchi/api-center-sample/blob/main/src/WebApp/Components/Pages/Home.razor) to consume the client SDK. Then you will be able to see the result.

![Pet Store - available pets][image-12]

Your web application as the API consumer works perfectly with the client SDK generated from APIC.

---

So far, I've walked through how [Azure API Center][apic] can handle your organisation's APIs as a central repository, and played around the APIC extension on VS Code. This post has shown you how to provision the APIC instance, register and import APIs in various ways, and how to test those APIs on VS Code and generate the client SDKs directly from VS Code.

As I mentioned in the beginning, taking care of many APIs in one place is crucial as your ogranisation grows up. You might think that you don't need APIC if your organisation's API structure is relatively simple. However, even if your organisation is small, APIC will give you better overview of APIs, and how they can interconnected with each other.

## More about Azure API Center?

If you want to learn more about APIC, the following links might be helpful.

- [Azure API Center][apic]
- [Create the first API Center][apic portal]
- [Playlist: Azure API Center][apic playlist]
- [Azure API Center Feedback][apic feedback]

[image-01]: /2024/02/api-center-first-look-01.png
[image-02]: /2024/02/api-center-first-look-02.png
[image-03]: /2024/02/api-center-first-look-03.png
[image-04]: /2024/02/api-center-first-look-04.png
[image-05]: /2024/02/api-center-first-look-05.png
[image-06]: /2024/02/api-center-first-look-06.png
[image-07]: /2024/02/api-center-first-look-07.png
[image-08]: /2024/02/api-center-first-look-08.png
[image-09]: /2024/02/api-center-first-look-09.png
[image-10]: /2024/02/api-center-first-look-10.png
[image-11]: /2024/02/api-center-first-look-11.png
[image-12]: /2024/02/api-center-first-look-12.png

[gh sample]: https://github.com/devkimchi/api-center-sample

[apic]: https://learn.microsoft.com/azure/api-center/overview?WT.mc_id=dotnet-122171-juyoo
[apic portal]: https://learn.microsoft.com/azure/api-center/set-up-api-center?WT.mc_id=dotnet-122171-juyoo
[apic portal register]: https://learn.microsoft.com/azure/api-center/register-apis?WT.mc_id=dotnet-122171-juyoo
[apic feedback]: https://github.com/Azure/api-center-preview
[apic playlist]: https://www.youtube.com/playlist?list=PLI7iePan8aH75Qz8h4yQBEC-uS339CUyi

[apim]: https://learn.microsoft.com/azure/api-management/api-management-key-concepts?WT.mc_id=dotnet-122171-juyoo

[az bicep]: https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview?tabs=bicep&WT.mc_id=dotnet-122171-juyoo
[az cli]: https://learn.microsoft.com/cli/azure/what-is-azure-cli?WT.mc_id=dotnet-122171-juyoo
[az cli extension apic]: https://github.com/Azure/azure-cli-extensions/tree/main/src/apic-extension

[azd cli]: https://learn.microsoft.com/azure/developer/azure-developer-cli/overview?WT.mc_id=dotnet-122171-juyoo

[vs code]: https://code.visualstudio.com?WT.mc_id=dotnet-122171-juyoo
[vs code extension apic]: https://marketplace.visualstudio.com/items?itemName=apidev.azure-api-center&WT.mc_id=dotnet-122171-juyoo
[vs code extension rest]: https://marketplace.visualstudio.com/items?itemName=humao.rest-client&WT.mc_id=dotnet-122171-juyoo
[vs code extension kiota]: https://marketplace.visualstudio.com/items?itemName=ms-graph.kiota&WT.mc_id=dotnet-122171-juyoo
