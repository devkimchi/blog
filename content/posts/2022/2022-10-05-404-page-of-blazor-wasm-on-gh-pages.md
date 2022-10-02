---
title: "Handling Custom 404 Page on GH Pages with Blazor WASM"
slug: 404-page-of-blazor-wasm-on-gh-pages
description: "In this post, I'm going to discuss how to handle the custom 404 page on GH Pages using Blazor WASM."
date: "2022-10-05"
author: Justin-Yoo
tags:
- dotnet
- blazor-wasm
- gh-pages
- 404-page
cover: /2022/10/404-page-of-blazor-wasm-on-gh-pages-00.png
fullscreen: true
---

It might be necessary to implement the custom 404 page while developing a [Blazor WebAssembly (WASM)][blazor wasm] app. But when you deploy your Blazor WASM app to [GitHub Pages][gh pages], it only shows GitHub's 404 page, not yours. Although there are many workarounds for this, throughout this post, I'm going to discuss how to use a Blazor web component for the 404 page on GitHub Pages.

> You can download the sample application on this [GitHub repository][gh sample].


## Blazor WASM &ndash; Default 404 Page ##

The default 404 page right after you create a Blazor WASM app project looks like this:

![Default 404 page][image-01]

It's because `App.razor` defines like that.

```razor
@* BlazorApp1.App.razor *@

<Router AppAssembly="@typeof(App).Assembly">
    ...
    <NotFound>
        <PageTitle>Not found</PageTitle>
        <LayoutView Layout="@typeof(MainLayout)">
            @* ⬇️⬇️⬇️ This is the content ⬇️⬇️⬇️*@
            <p role="alert">Sorry, there's nothing at this address.</p>
            @* ⬆️⬆️⬆️ This is the content ⬆️⬆️⬆️*@
        </LayoutView>
    </NotFound>
</Router>
```

Therefore, all you need to customise your 404 page is this part.


## Blazor WASM &ndash; Custom 404 Page ##

The sample app used in this post uses a customised 404 page. For the custom 404 page, create a web component called `NotFound.razor`.

```razor
@* Fitability.Home.Components.NotFound.razor *@

<div class="row">
    <div class="col-6">
        <img src="images/banner-3840x1920.png" class="img-fluid" alt="banner" />
    </div>
    <div class="col-6 d-flex align-items-center">
        <div class="row">
            <div class="col-12">
                <h1 class="text-center">Not Found!</h1>
                <p class="text-center fs-2">Your requested page doesn't exist.</p>
                <p class="text-center fs-3"><a href="/" class="btn btn-primary btn-lg">Home</a></p>
            </div>
        </div>
    </div>
</div>
```

Then, import it within the `LayoutView` component of `App.razor`

```razor
@* Fitability.Home.App.razor *@

<Router AppAssembly="@typeof(App).Assembly">
    ...
    <NotFound>
        <LayoutView Layout="@typeof(MainLayout)">
            @* ⬇️⬇️⬇️ Add the NotFound component here ⬇️⬇️⬇️*@
            <NotFound />
            @* ⬆️⬆️⬆️ Add the NotFound component here ⬆️⬆️⬆️*@
        </LayoutView>
    </NotFound>
</Router>
```

Run your Blazor WASM app on your local machine, and you will see the custom 404 page.

![Custom 404 page][image-02]


## GitHub Pages &ndash; Default 404 Page ##

However, deploy your Blazor WASM app to GitHub Pages and visit any non-existing URL. Then you will see GitHub's default 404 page.

![GitHub default 404 page][image-03]

It's because GitHub displays their default 404 page if someone visits the non-existing page. Let's change this.


## GitHub Pgaes &ndash; Custom 404 Page ##

The `404.html` file MUST physically exist to use the custom 404 page on your GitHub Pages. Generally speaking, you can prepare hard-coded `404.html` for this purpose. But we're using the Blazor WASM's routing pipeline.

1. Copy the existing `index.html` file to `404.html`. Both are fundamentally the same file as each other.
2. Create the `404.razor` page and import the `NotFound` component. The routing path of this `404.razor` page MUST be `/404.html`.

    ```razor
    @* Fitability.Home.Pages.404.razor *@

    @page "/404.html"

    <NotFound />
    ```

Deploy your Blazor WASM app to GitHub Pages again and visit any non-existing URL. Then, you will be able to see your custom 404 page.

![Custom 404 page on GH Pages][image-04]

It's because GitHub Pages looks for the physical `404.html` file. This file runs on the Blazor WASM routing pipeline. Therefore, you can write any business logic on your 404 page.

---

So far, we've walked through how to deal with the custom 404 page of the [Blazor WASM][blazor wasm] app running on [GitHub Pages][gh pages].


## Want to know more about Blazor? ##

It's great if you visit those sites for more Blazor stuff.

* [Blazor][blazor]
* [Blazor Tutorials][blazor tutorial]
* [Blazor Learn][blazor learn]


[image-01]: /2022/10/404-page-of-blazor-wasm-on-gh-pages-01.png
[image-02]: /2022/10/404-page-of-blazor-wasm-on-gh-pages-02.png
[image-03]: /2022/10/404-page-of-blazor-wasm-on-gh-pages-03.png
[image-04]: /2022/10/404-page-of-blazor-wasm-on-gh-pages-04.png


[gh sample]: https://github.com/fitability/fitability.github.io
[gh pages]: https://pages.github.com/

[blazor]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor?WT.mc_id=dotnet-77749-juyoo
[blazor tutorial]: https://dotnet.microsoft.com/learn/aspnet/blazor-tutorial/intro?WT.mc_id=dotnet-77749-juyoo
[blazor learn]: https://learn.microsoft.com/training/paths/build-web-apps-with-blazor/?WT.mc_id=dotnet-77749-juyoo
[blazor wasm]: https://learn.microsoft.com/aspnet/core/blazor/?WT.mc_id=dotnet-77749-juyoo#blazor-webassembly
