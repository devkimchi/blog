---
title: "Blazor Code-behind"
slug: blazor-code-behind
description: "This post shows how to separate the C# code block from a .razor file in a Blazor application to comply with the separation of concerns principle."
date: "2021-03-10"
author: Justin-Yoo
tags:
- blazor-server
- blazor-webassembly
- code-behind
- separation-of-concerns
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/03/blazor-code-behind-00.png
fullscreen: true
---

Suppose you've got an [ASP.NET WebForm][aspnet webform] application that has been running for ages. As it's only run on the [.NET Framework][dotnet framework], and there will be no more new feature added, Microsoft recommends migrating the app to [.NET 5][dotnet 5]. However, for many reasons, migration wouldn't be that easy. Despite those reasons, what could you do for migration from ASP.NET WebForm to .NET 5? There are several alternatives, but the most pragmatic and feasible way that might be able to minimise the migration cost is to move to Blazor.

When you see the `Counter.razor` file as soon as a new [Blazor app][blazor] is created, it looks like:

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=01-counter.razor

As you can see, both the HTML part and code part co-exist in one single `.razor` file. It's OK to run the app like that as long as the app works fine. But as the business grows, requirements get complicated, and the size gets bigger, splitting the `.razor` file into two &ndash; HTML document and its code-behind &ndash; would become more beneficial. In fact, modern app development also strongly recommends applying the [Separation of Concerns][oop soc] principle. Throughout this post, I'm going to discuss how to divide the `.razor` file into two different files.

> * This post is based on the doc, [ASP.NET Core Razor Components &ndash; Partial Class Support][blazor components partial].
> * The term, Blazor, points to both [Blazor Server][blazor server] and [Blazor Web Assembly][blazor wasm] in general unless specifically mentioned.
> * You can download the sample code from this [GitHub repository][gh sample].


## Building Partial Class ##

Basically, all `.razor` files become the class objects for themselves. In other words, the `Counter.razor` file mentioned above becomes the `Counter` class once compiled. Therefore, to separate HTML and code from each other, the code-behind must use the [`partial` modifier][dotnet partial] when declaring a class (line #1).

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=02-counter.cs&highlights=1

Then move the codes inside the `@code` block to the `Counter` class.

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=03-counter.cs

Compile the app and run it, and you will see the Blazor app is up and running without an issue.


## Inheriting Component Model ##

Like above, you can use the `partial` modifier. There is another approach, inheriting a component model class. It brings up more benefits because the component model already provides properties and events which you can utilise. Create a `CounterBase` class by inheriting the `ComponentBase` class (line #1).

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=04-counterbase.cs&highlights=1

Then, add the `@inherits` directive into the `Counter.razor` file (line #2):

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=05-counter.razor&highlights=2

You will find out the red curly underlines on both `@currentCount` and `IncrementCount` that indicate errors.

![Error on Counter.razor][image-01]

It's because both `currentCount` field `IncrementCount()` method are in the scope of `private`. To access both, the scoping must be changed to `protected` at least. Change the scope from `private` to `protected` and update the `currentCount` field to the `CurrentCount` property (line #3, 5).

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=06-counterbase.cs&highlights=3,5

Let's add one more thing here. The `ComponentBase` class contains various methods impacting the component's lifecycle. The `OnInitializedAsync()` method is one of them. When you look at the method, as a WebForm developer, it looks very similar to either the `Page_Init()` or `Page_Load()` method. Let's initiate the `CurrentCount` property value to `10` (line #10-13).

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=07-counterbase.cs&highlights=10-13

Then, update the `Counter.razor` file to refer to the protected property of `CurrentCount` (line #6) instead of the private field of `currentCount`. Those red curly underlines have disappeared!

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=08-counter.razor&highlights=6

Compile the app and rerun it, and confirm the app works as expected. In addition to that, you will see the `Current count` value of `10` by default.

![Initialised to 10 at Counter.razor][image-02]


## Combination of Partial Class and ComponentBase Inheritance ##

The downside of using the component model is to use the `@inherits` directives within the accompanying `.razor` file. It might be small, but to me, that seems redundant. Can I avoid using the directive? Of course, I can! Just use the `partial` modifier to the inheriting class like below. Change the class name to `Counter` and add the `partial` modifier (line #1).

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=09-counter.cs&highlights=1

Then, remove the `@inherits` directive from the `Counter.razor` file.

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=10-counter.razor

Let's compile the app and run it. Can you see what you expected?

---

So far, we've walked through how to extract the codes from HTML in a `.razor` file of a Blazor application. In fact, it's not a tip nor a trick, but a suggestion for a better app development approach. Suppose you or your organisation plans to migrate existing [ASP.NET WebForm][aspnet webform] applications to [Blazor][blazor] and wants to keep the same development experience. In that case, this post will be the starting point.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/03/blazor-code-behind-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/03/blazor-code-behind-02.png

[gh sample]: https://github.com/devkimchi/Blazor-Code-Behind-Sample

[oop soc]: https://docs.microsoft.com/dotnet/architecture/modern-web-apps-azure/architectural-principles?WT.mc_id=dotnet-19728-juyoo#separation-of-concerns

[dotnet framework]: https://dotnet.microsoft.com/download/dotnet-framework?WT.mc_id=dotnet-19728-juyoo
[dotnet 5]: https://dotnet.microsoft.com/download/dotnet/5.0?WT.mc_id=dotnet-19728-juyoo
[dotnet partial]: https://docs.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/partial-classes-and-methods?WT.mc_id=dotnet-19728-juyoo

[aspnet webform]: https://docs.microsoft.com/aspnet/web-forms/what-is-web-forms?WT.mc_id=dotnet-19728-juyoo
[aspnet webform codebehind]: https://docs.microsoft.com/troubleshoot/aspnet/code-behind-model?WT.mc_id=dotnet-19728-juyoo

[blazor]: https://docs.microsoft.com/aspnet/core/blazor/?view=aspnetcore-5.0&WT.mc_id=dotnet-19728-juyoo
[blazor server]: https://docs.microsoft.com/aspnet/core/blazor/hosting-models?view=aspnetcore-5.0&WT.mc_id=dotnet-19728-juyoo#blazor-server
[blazor wasm]: https://docs.microsoft.com/aspnet/core/blazor/hosting-models?view=aspnetcore-5.0&WT.mc_id=dotnet-19728-juyoo#blazor-webassembly
[blazor components partial]: https://docs.microsoft.com/aspnet/core/blazor/components/?view=aspnetcore-5.0&WT.mc_id=dotnet-19728-juyoo#partial-class-support
