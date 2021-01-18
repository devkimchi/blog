---
title: "Dev.To Article Publish Scheduler"
slug: devto-article-publish-scheduler
description: "This post shows how to build a scheduling application for dev.to, using Azure Durable Functions."
date: "2021-01-20"
author: Justin-Yoo
tags:
- azure-functions
- azure-durable-functions
- devto
- frontmatter
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/01/devto-article-publish-scheduler-00-en.png
fullscreen: true
---

There's the tool called [PublishToDev][todd publishtodev] built by one of my colleagues, [Todd][todd], which schedules to publish articles on [Dev.To][devto]. It's super useful because I can schedule my posts whenever I want to publish them on there. As soon as I saw this tool, I wanted to clone code in .NET because it would be beneficial to practice:

* Scraping web pages using either [Puppeteer][pptr] or [Playwright][pwr],
* Using the [Dev.To API][devto api],
* Handling the frontmatter programmatically, and
* Writing [Azure Durable Functions][az fncapp durable]

Let's walk through how I made it.

> You can find the entire source codes of this application at [this GitHub repository][gh sample].


## Web Pages Scraping ##

Once you write a blog post on [Dev.To][devto], you'll be able to get a preview URL before it publishing it. The preview URL has the format of `https://dev.to/<username>/xxxx-****-temp-slug-xxxx?preview=xxxx`. All you need to know from the preview page is to get the article ID found from the HTML element having the attribute of `id="article-body"`.

![HTML document view on Dev.To preview page][image-01]

According to the picture above, you can find the attribute of `data-article-id`. Its value is the very article ID.

Using either [Puppeteer][pptr] or [Playwright][pwr] to scrape a web page is super simple. Both have their own .NET ported versions like [Puppeteer Sharp][pptr csharp] and [Playwright Sharp][pwr csharp] respectively. However, they don't work on [Azure Functions][az fncapp], unfortunately. More precisely, they work on your local dev environment, not on Azure instance. [This post][pwr anthony] would be useful for your node.js Azure Functions app, but it's not that helpful for your .NET application. *Let me find a way for it to work on Azure Functions instance correctly.*

Therefore, I had to change the scraping method to be a traditional way, using [`HttpClient`][dotnet core httpclient] and regular expressions (line #1-2, 8).

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=01-scraping-article-id.cs&highlights=1-2,8

[![The ends justifies the means. - Niccolo Machiavelli](https://www.azquotes.com/picture-quotes/quote-the-ends-justifies-the-means-niccolo-machiavelli-40-57-73.jpg)](https://www.azquotes.com/quote/405773)

You've got the article ID of your post. Let's move on.


## Dev.To API Document &ndash; Open API ##

[Dev.To][devto] is a blog platform for developer communities. Tens of blog posts with a broad range of development topics are published day by day. It also provides APIs to publish and manage blog posts. In other words, it has well [documented APIs][devto api]. Within the document page, you can also find the Open API document, which you will be able to build a wrapper SDK instantly.


### Wrapper SDK Generation with AutoRest ###

As long as you've got an Open API document, generating an SDK is a piece of cake, using [AutoRest][autorest]. I created a .NET SDK by the following command. I set the namespace of `Aliencube.Forem.DevTo` and output directory of `output`. The last `--v3` option indicates that the Open API document conforms to the v3 spec version.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=02-generate-sdk.sh

[AutoRest][autorest] does not only generate SDK in .NET but also in Go, Java, Python, node.js, TypeScript, Ruby and PHP. Therefore, you can generate the SDK with your desired language. The wrapper SDK repository can be found at:

[https://github.com/aliencube/forem-sdk][devto api wrapper]


## Blog Post Markdown Document Download ##

To use the API, you need to have an API key, of course. In the [account settings page][devto account], generate a new API key.

![Dev.To API Key][image-02]

Then, use the wrapper SDK generated above, and you'll get the markdown document (line #4-6).

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=03-get-markdown.cs&highlights=4-6


## Frontmatter Update ##

All the blog posts published to Dev.To contain metadata called frontmatter at the top of the markdown document. The frontmatter is written in YAML. Your blog post markdown might look like:

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=04-frontmatter.yaml

In the frontmatter, you'll see the key/value pair of `published: false`. Updating this value to `true` and saving the post means that your blog post will be published. Therefore, all you need to do is to update that value in the frontmatter area. Have a look at the code below, which extracts the frontmatter from the markdown document.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=05-extract-frontmatter.cs

The frontmatter string needs to be deserialised to a strongly-typed `FrontMatter` instance, using the [YamlDotNet][ydn] library. Then, change the `Published` value to `true`.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=06-deserialise-frontmatter.cs

Once updated the frontmatter instance, serialise it again and concatenate it with the existing markdown body.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=07-update-markdown.cs


## Blog Post Markdown Document Update ##

Now, make another API call with this updated markdown document, and your post will be published.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=08-update-article.cs

This is how your Dev.To blog post is published via their API. Let's move onto the scheduling part.


## Azure Durable Functions for Scheduling ##

It's good to understand that [Azure Durable Functions][az fncapp durable] is a combination of three unit functions&ndash;[API endpoint function or durable client function][az fncapp durable client], [orchestrator function][az fncapp durable orchestrator] and [activity function][az fncapp durable activity]. Each has its respective role in the following scenarios.

1. The API endpoint function accepts the API requests. It then calls the orchestrator function to manage the entire workflow and returns a response with the 202 status code.
2. The orchestrator function controls when and how activity functions are called, and aggregate states.
3. Individual activity functions do their jobs and share the result with the orchestrator function.

![Azure Durable Functions Workflow][image-03]

The orchestrator function also includes the [timer feature][az fncapp durable timer] as one of the controlling methods for activity functions. With this timer, we can do the scheduling. In other words, we temporarily save the blog post at one time, then schedule to publish it by setting a timer.


### API Endpoint Function ###

[The endpoint function][az fncapp durable client] is the only type to be exposed outside. It's basically the same as the [HTTP trigger function][az fncapp trigger http], but it has additional parameter with the [durable function binding][az fncapp durable binding] (line #4).

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=09-api-endpoint-function-1.cs&highlights=4

What does it do, by the way?

1. The function accepts API requests from outside, with a request payload. In this post, the request payload looks like the following JSON object. The `schedule` value should follow the [ISO8601][iso8601] format (eg. `2021-01-20T07:30:00+09:00`).

    https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=10-api-request-payload.json

2. Deserialise the request payload.

    https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=09-api-endpoint-function-2.cs

3. Create a new orchestrator function and call it with the request payload.

    https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=09-api-endpoint-function-3.cs

4. As the orchestrator function works asynchronously, the endpoint function responds with the HTTP status code of 202.

    https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=09-api-endpoint-function-4.cs


### Orchestrator Function ###

The [orchestrator function][az fncapp durable orchestrator] takes care of the entire workflow. Here's the binding for the orchestrator function (line #3).

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=10-orchestraction-function-1.cs&highlights=3

`IDurableOrchestrationContext` instance knows the request payload passed from the endpoint function.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=10-orchestraction-function-2.cs

Activate a timer, using the schedule from the request payload.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=10-orchestraction-function-3.cs

Once the timer is activated, the orchestrator function is suspended until the timer expires. Once the timer expires, the orchestrator function resumes and calls the activity function.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=10-orchestraction-function-4.cs

Finally, it returns the result aggregated from the activity function.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=10-orchestraction-function-5.cs


## Activity Function ##

While both endpoint function and orchestrator function do not look after the blog post itself, the [activity function][az fncapp durable activity] does all the things, including web page scraping, Dev.To API call and markdown document update. Here's the binding for the activity function (line #3).

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=11-activity-function-1.cs&highlights=3

Add the codes for scraping, API call and markdown update mentioned above.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=11-activity-function-2-en.cs

And, it finally returns the result.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=11-activity-function-3.cs

---

So far, we've walked through implementing an [Azure Durable Functions app][az fncapp durable] to schedule to publish articles to the Dev.To platform. Throughout this, I think you've understood the core workflow of Azure Durable Functions&ndash;API request, orchestration and individual activities. The power of the Durable Functions is that it overcomes the limitations of stateless, by storing states. I hope you feel this power and convenience, too.


[image-01]: https://user-images.githubusercontent.com/1538528/89647854-3a9c4e80-d8f9-11ea-8415-a71aac67168c.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/01/devto-article-publish-scheduler-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/01/devto-article-publish-scheduler-03-en.png

[todd]: https://twitter.com/toddanglin
[todd publishtodev]: https://github.com/toddanglin/PublishToDev

[gh sample]: https://github.com/devrel-kr/devto-article-publish-scheduler

[pptr]: https://pptr.dev/
[pptr csharp]: https://www.puppeteersharp.com/

[pwr]: https://playwright.dev/
[pwr node]: https://www.npmjs.com/package/playwright
[pwr csharp]: https://github.com/microsoft/playwright-sharp
[pwr anthony]: https://anthonychu.ca/post/azure-functions-headless-chromium-puppeteer-playwright/

[dotnet core httpclient]: https://docs.microsoft.com/dotnet/api/system.net.http.httpclient?view=netcore-3.1&WT.mc_id=dotnet-12868-juyoo

[devto]: https://dev.to/
[devto account]: https://dev.to/settings/account
[devto api]: https://docs.forem.com/api/
[devto api wrapper]: https://github.com/aliencube/forem-sdk

[autorest]: https://github.com/Azure/autorest

[ydn]: https://github.com/aaubry/YamlDotNet

[iso8601]: https://en.wikipedia.org/wiki/ISO_8601

[az fncapp]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-12868-juyoo
[az fncapp trigger http]: https://docs.microsoft.com/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=csharp&WT.mc_id=dotnet-12868-juyoo
[az fncapp durable]: https://docs.microsoft.com/azure/azure-functions/durable/durable-functions-overview?tabs=csharp&WT.mc_id=dotnet-12868-juyoo
[az fncapp durable timer]: https://docs.microsoft.com/azure/azure-functions/durable/durable-functions-timers?tabs=csharp&WT.mc_id=dotnet-12868-juyoo
[az fncapp durable binding]: https://docs.microsoft.com/azure/azure-functions/durable/durable-functions-bindings?WT.mc_id=dotnet-12868-juyoo
[az fncapp durable query]: https://docs.microsoft.com/azure/azure-functions/durable/durable-functions-instance-management?tabs=csharp&WT.mc_id=dotnet-12868-juyoo
[az fncapp durable client]: https://docs.microsoft.com/azure/azure-functions/durable/durable-functions-types-features-overview?WT.mc_id=dotnet-12868-juyoo#client-functions
[az fncapp durable orchestrator]: https://docs.microsoft.com/azure/azure-functions/durable/durable-functions-types-features-overview?WT.mc_id=dotnet-12868-juyoo#orchestrator-functions
[az fncapp durable activity]: https://docs.microsoft.com/azure/azure-functions/durable/durable-functions-types-features-overview?WT.mc_id=dotnet-12868-juyoo#activity-functions
