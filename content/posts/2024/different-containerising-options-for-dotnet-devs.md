---
title: "Different Containerising Options for .NET Developers"
slug: different-containerising-options-for-dotnet-devs
description: "Throughout this post, I'm going to discuss which options are available for .NET developers to containerise our .NET apps."
date: "2024-08-12"
author: Justin-Yoo
tags:
- dotnet
- containers
- docker
cover: /2024/08/different-containerising-options-for-dotnet-devs-00.png
fullscreen: true
---

As a .NET developer, if you want to containerise your .NET apps, you know you need a `Dockerfile` first. `Dockerfile` is a spec that describes which OS is used, where the .NET code is located, how the .NET code is compiled and how the .NET app is executed. Once your Dockerfile is ready, you can build your container image. But is the Dockerfile the only option for building your .NET apps in a container? How about the container orchestration? Is there any way to orchestrate containers other than running the `docker compose up` command? Throughout this post, I'm going to discuss which options are available for .NET developers to containerise your .NET apps.

> You can find a sample code from this [GitHub repository][gh sample].

## Prerequisites

There are a few prerequisites to containerise .NET apps effectively.

- [.NET SDK 8.0+][dotnet sdk] with [.NET Aspire workload][dotnet aspire workload] and [Aspirate][dotnet aspire aspirate]
- [Docker Desktop][docker desktop]

## Containerise with `Dockerfile`

In the sample code repository, there are two .NET apps, `ApiApp` and `WebApp`. When you run both apps locally, it communicates with each other by running the following commands:

```powershell
dotnet run --project ./MSBuildForContainers.ApiApp
```

```powershell
dotnet run --project ./MSBuildForContainers.WebApp
```

Now, you want to containerise both apps. How can you do that? The first option is to write a `Dockerfile` for each app. You can either manually write `Dockerfile` or run the command, `docker init`. Once you have `Dockerfile` files for both apps, run the following commands to build the container images:

```powershell
# For ApiApp
docker build . -t apiapp:latest

# For WebApp
docker build . -t webapp:latest
```

We all know how `Dockerfile` is useful to build container images. However, `Dockerfile` is yet another code and should be maintained. If you have many apps to containerise, you need to write `Dockerfile` files respectively. This is a bit cumbersome. Is there any other way to containerise .NET apps, without writing `Dockerfile` files?

## Containerise with `dotnet publish`

MSBuild supports containerisation by itself. It uses the `dotnet publish` command to build the container image for each app. To do this, you might need to update your `.csproj` file to include the containerisation settings. The following is the `MSBuildForContainers.ApiApp.csproj` file:

```xml
<PropertyGroup>
  <ContainerRepository>apiapp</ContainerRepository>
  <ContainerImageTag>latest</ContainerImageTag>
</PropertyGroup>
```

These two properties replaces the `--tag` option in the `docker build` command. Once you have these properties in the `.csproj` file, run the following command. Note that the `-t:PublishContainer` indicates the containerisation and the `--os` and `--arch` options specify the target OS and architecture.

```powershell
dotnet publish ./MSBuildForContainers.ApiApp \
    -t:PublishContainer \
    --os linux --arch x64
```

Then, you'll have the container image. If you want to change the base image to the [chiseled container one][dotnet container chiseled], add another property to the `.csproj` file. The following property sets the chiseled Ubuntu 24.04 image as the base image.

```xml
<PropertyGroup>
  <ContainerBaseImage>mcr.microsoft.com/dotnet/aspnet:8.0-noble-chiseled</ContainerBaseImage>
</PropertyGroup>
```

Then, run the `dotnet publish` command above again. This time, the container image is built based on the chiseled image. Let's do the same thing against the `MSBuildForContainers.WebApp.csproj` file. This way, you can containerise your .NET apps without writing `Dockerfile` files.

Alternatively, you don't even need those properties in the `.csproj` file. You can run the following command to build the container image directly like this:

```powershell
dotnet publish ./MSBuildForContainers.ApiApp \
    -t:PublishContainer \
    --os linux --arch x64 \
    -p:ContainerBaseImage=mcr.microsoft.com/dotnet/aspnet:8.0-noble-chiseled \
    -p:ContainerRepository=apiapp \
    -p:ContainerImageTag=latest
```

You have the same development experience as building the container image with the `docker build` command, without having to rely on `Dockerfile`.

Now, you've got chiseled container images for both API app and web app. How to let them talk to each other?

## Container orchestration with `docker compose`

The existing web app container image can't talk to the API app container image because it doesn't know where the API app container is. To make the web app container talk to the API app container, you need to slightly modify the web app and build the container image again. Open the `MSBuildForContainers.WebApp/Program.cs` file and update the base address value as follows:

```csharp
// Before
builder.Services.AddHttpClient<IApiAppClient, ApiAppClient>(http => http.BaseAddress = new Uri("https://localhost:5051/"));

// After
builder.Services.AddHttpClient<IApiAppClient, ApiAppClient>(http => http.BaseAddress = new Uri("http://apiapp:8080/"));
```

Run the `dotnet publish` command again to build the web app container image. Now, you can manually run the `docker run` command to run both containers by attaching the same network. But for the container orchestration, the `docker compose up` command is much easier to use. The `docker-compose.yml` file is already prepared in the sample code repository. Run the following command:

```powershell
docker compose -f ./docker-compose.yaml up
```

Then, both containers are up and running. The web app container can talk to the API app container. This is how you can orchestrate containers with `docker compose up`. Again, this Docker Compose file is yet another code and should be maintained. If you have many apps to orchestrate, how would you manage them?

## Container orchestration with .NET Aspire and Aspirate

[.NET Aspire][dotnet aspire] is to orchestrate .NET apps in containers. For this Docker Compose orchestration purpose, [Aspirate][dotnet aspire aspirate] is used, which is a tool that generates the Docker Compose file, a [Kustomize file][k8s kustomize] or [helm file][k8s helm] for the container orchestration. To use Aspirate, you need to install it first:

```powershell
dotnet tool install -g aspirate
```

In the sample code repository, Both web app and API app are orchestrated with .NET Aspire in another branch. Switch to the `aspire` branch with the command:

```powershell
git switch aspire
```

Let's generate the Docker Compose file with Aspirate. Run the following command to generate the .NET Aspire manifest file first:

```powershell
dotnet run --project ./MSBuildForContainers.AppHost \
    -- \
    --publisher manifest \
    --output-path ../aspire-manifest.json
```

Then, generate the Docker Compose file with the Aspirate command. Note that it intentionally excludes the .NET Aspire dashboard.

```powershell
aspirate generate \
    --project-path ./MSBuildForContainers.AppHost \
    --aspire-manifest ./aspire-manifest.json \
    --output-format compose \
    --disable-secrets --include-dashboard false
```

Now, you have the Docker Compose file generated. Run the `docker compose up` command to orchestrate the containers. This way, you can orchestrate your .NET apps in containers without manually writing the Docker Compose file.

---

So far, I've walked through how .NET developers can containerise their .NET apps with different options. You can either write `Dockerfile` files or use `dotnet publish` to build container images. You can orchestrate containers by writing the Docker Compose file by hand or letting .NET Aspire and Aspirate generate it automatically. Which one do you prefer?

## More about MSBuild for Containers?

If you want to learn more options about containers with MSBuild, the following links might be helpful.

- [.NET Aspire][dotnet aspire]
- [Containerise a .NET app with `dotnet publish`][dotnet publish container]
- [.NET container images][dotnet container images]

[gh sample]: https://github.com/devkimchi/msbuild-for-containers

[dotnet sdk]: https://dotnet.microsoft.com/download/dotnet/8.0?WT.mc_id=dotnet-147226-juyoo
[dotnet aspire]: https://learn.microsoft.com/dotnet/aspire/get-started/aspire-overview?WT.mc_id=dotnet-147226-juyoo
[dotnet aspire workload]: https://learn.microsoft.com/dotnet/aspire/fundamentals/setup-tooling?WT.mc_id=dotnet-147226-juyoo
[dotnet aspire aspirate]: https://github.com/prom3theu5/aspirational-manifests
[dotnet publish container]: https://learn.microsoft.com/dotnet/core/docker/publish-as-container?WT.mc_id=dotnet-147226-juyoo
[dotnet container images]: https://learn.microsoft.com/dotnet/core/docker/container-images?WT.mc_id=dotnet-147226-juyoo
[dotnet container chiseled]: https://devblogs.microsoft.com/dotnet/announcing-dotnet-chiseled-containers/?WT.mc_id=dotnet-147226-juyoo

[docker desktop]: https://docs.docker.com/desktop/

[k8s kustomize]: https://kustomize.io/
[k8s helm]: https://helm.sh/
