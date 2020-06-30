---
title: "5 Ways Selecting Injected Instance from Multiple Instances of Same Interface on ASP.NET Core"
slug: 5-ways-injecting-multiple-instances-of-same-interface-on-aspnet-core
description: "This post shows how to implement a logic that selects an injected instance from multiple instances of the same interface on ASP.NET Core."
date: "2020-07-01"
author: Justin-Yoo
tags:
- aspnet-core
- dependency-injection
- ioc-container
- multiple-instances
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2020/07/5-ways-injecting-multiple-instances-of-same-interface-on-aspnet-core-00.png
fullscreen: true
---

While building an [ASP.NET Core][aspnet core] application, [setting an IoC container for dependency injection][aspnet core di] is nearly inevitable. ASP.NET Core offers a built-in IoC container that is easy to use, without having to rely on any third-party libraries. Throughout this post, I'm going to discuss five different ways to pick up a dependency injected from multiple instances sharing with the same interface.


## Implementing Interface and Classes ##

Let's assume that we're writing one `IFeedReader` interface and three different classes, `BlogFeedReader`, `PodcastFeedReader` and `YouTubeFeedReader` implementing the interface (line #1, 7, 22, 37). There's nothing fancy.

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=01-feed-reader.cs&highlights=1,7,22,37


## IoC Container Registration #1 &ndash; Collection ##

Let's register all the classes declared above into the `ConfigureServices()` method in `Startup.cs`.

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=02-configure-services.cs

By doing so, all three classes have been registered as `IFeedReader` instances. However, from the consumer perspective, it doesn't know which instance is appropriate to use. In this case, using a collection as `IEnumerable<IFeedReader>` is useful. In other words, inject a dependency of `IEnumerable<IFeedReader>` to the `BlogFeedService` class and filter one instance from the collection (line #7).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=03-feed-service.cs&highlights=7

It's one way to use the collection as a dependency. The other way to use the collection is to use a loop (line #12). It's useful when we implement either a [Visitor Pattern][design pattern visitor] or [Iterator Pattern][design pattern iterator].

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=04-feed-service.cs&highlights=12


## IoC Container Registration #2 &ndash; Resolver ##

It's similar to the first approach. This time, let's use a resolver instance to get the dependency we want. First of all, declare both `IFeedReaderResolver` and `FeedReaderResolver` (line #1, 6). Keep an eye on the instance of `IServiceProvider` as a dependency. It's used for the built-in IoC container of ASP.NET Core, which can access to all registered dependencies.

In addition to that, this time, we don't need the `Name` property any longer as we use conventions to get the required instance (line 17-18).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=05-feed-reader-resolver.cs&highlights=1,6,17-18

After that, update `ConfigureServices()` on `Startup.cs` again. Unlike the previous approach, we registered `xxxFeedReader` instances, not `IFeedReader`. It's fine, though. The resolver sorts this out for `BlogFeedService`.

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=06-configure-services.cs

Update the `BlogFeedService` class like below (line #5).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=07-feed-service.cs&highlights=5


## IoC Container Registration #3 &ndash; Resolver + Factory Method Pattern ##

Let's slightly modify the resolver that uses the factory method pattern. After this modification, it removes the dependency on the `IServiceProvider` instance. Instead, it creates the required instance by using the `Activator.CreateInstance()` method (line #5-6).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=08-feed-reader-resolver.cs&highlights=5-6

If we implement the resolver class in this way, we don't need to register all `xxxFeedReader` classes to the IoC container, but `IFeedReaderResolver` would be sufficient. By the way, make sure that all `xxxFeedReader` instances cannot be used as a singleton if we take this approach.

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=09-configure-services.cs


## IoC Container Registration #4 &ndash; Explicit Delegates ##

We can replace the resolver with an explicit [delegates][dotnet delegates]. Let's have a look at the code below. Within `Startup.cs`, declare a delegate just outside the `Startup` class.

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=10-feed-reader-delegate.cs

Then, update `ConfigureServices()` like below. As we only declared the delegate, its actual implementation goes here. The implementation logic is not that different from the previous approach (line #9-10).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=11-configure-services.cs&highlights=9-10

Update the `BlogFeedService` class (line #5). As `FeedReaderDelegate` returns the `IFeedReader` instance, another method call should be performed through the method chaining (line #12).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=12-feed-service.cs&highlights=5,12


## IoC Container Registration #5 &ndash; Implicit Delegates with Lambda Function ##

Instead of using the explicit delegate, the Lambda function can also be used as the implicit delegate. Let's modify the `ConfigureServices()` method like below. As there is no declaration of the delegate, define the Lambda function (line #7-13).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=13-configure-services.cs&highlights=7-13

As the injected object is the Lambda function, `BlogFeedService` should accept the Lambda function as a dependency (line #5). While it's closely similar to the previous example, it uses the Lambda function this time for dependency injection.

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=14-feed-service.cs&highlights=5

---

So far, we have discussed five different ways to resolve injected dependencies using the same interface in an [ASP.NET Core][aspnet core] application. Those five approaches are very similar to each other. Then, which one to choose? Well, there's no one approach better than the other four, but I guess it would depend purely on the developer's preference.

[dotnet delegates]: https://docs.microsoft.com/dotnet/csharp/programming-guide/delegates/?WT.mc_id=devkimchicom-blog-juyoo

[aspnet core]: https://docs.microsoft.com/aspnet/core/?view=aspnetcore-3.1&WT.mc_id=devkimchicom-blog-juyoo
[aspnet core di]: https://docs.microsoft.com/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-3.1&WT.mc_id=devkimchicom-blog-juyoo

[design pattern visitor]: https://www.dofactory.com/net/visitor-design-pattern
[design pattern iterator]: https://www.dofactory.com/net/iterator-design-pattern
