---
title: "GitHub Copilot for Azure API Management Policies"
slug: gh-copilot-for-apim-policies
description: "Throughout this post, I'm going to discuss how easy GitHub Copilot is to use for Azure API Management policy document management."
date: "2023-07-31"
author: Justin-Yoo
tags:
- azure
- api-management
- github-copilot
- azure-openai
cover: /2023/07/gh-copilot-for-apim-policies-00.png
fullscreen: true
---

[Azure API Management (APIM)][apim] is a tool that manages various types of backend APIs of your organisation. It offers many awesome features, and [APIM Policies][apim policies] is one of the ones. For example, You can use the APIM policies to extends the APIM features or configure security policies to protect your backend APIs. While you need to write the policy documents in the XML format, it has a fairly bit amount of learning curves &ndash; it's not that easy to use.

Throughout this post, I'm going to discuss how [GitHub Copilot][gh copilot] helps us write the APIM policy documents, with a few technical scenarios.

## APIM Policy documents at various levels

First of all, as soon as you provisions a new APIM instance, you'll see the default global policy document as follows:

```xml
<policies>
    <inbound />
    <backend>
        <forward-request />
    </backend>
    <outbound />
    <on-error />
</policies>
```

![APIM Default Policy Document &ndash; Global level][image-01]

Also, each API has its own default policy document as follows:

```xml
<policies>
    <inbound>
        <base />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

![APIM Default Policy Document &ndash; API level][image-02]

Likewise, each operation has its own default policy document as follows:

```xml
<policies>
    <inbound>
        <base />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

![APIM Default Policy Document &ndash; Operation level][image-03]

Once you get these default policy document, you need to define your own policy document based on your business logic. At this point, you can use the policy snippets in the picture below.

![APIM Policy Snippets][image-04]

However, it's still cumbersome to find the right snippet and insert it into the policy document. Also, it has a fairly bit amount of learning curves. To overcome this, let's use [GitHub Copilot][gh copilot]. As of writing this post, [GitHub Copilot Chat][gh copilot chat beta] is now available as a public beta version, which is a good timing.

## Prerequisits

- [Visual Studio Code][vs code]
- [GitHub Copilot subscription][gh copilot subscription] &ndash; If you are a student, join the [Student Developer Pack][gh student dev pack] program for free GitHub Copilot offer and other perks.
- [GitHub Copilot extension][vs code extensions copilot]
- [GitHub Copilot Chat extension][vs code extensions copilot chat]

## APIM Policy documents with GitHub Copilot

You can write APIM policy documents at the global, API and operation levels, and the context is slightly different. Let's take a look.

### Global-level policy document

First of all, let's write a global policy document. Here's the scenario:

> In most cases, you apply the CORS policy between the frontend and backend applications at the global level. Let's apply this CORS policy to the global policy document.

Open Visual Studio Code and open the GitHub Copilot Chat window.

![GitHub Copilot Chat &ndash; Window][image-05]

Enter the following [zero-shot prompt][aoai prompt-engineering]:

```yaml
Show me the Azure API Management policy document at the global level, including the following.

- CORS origins: https://make.powerapps.com, https://make.powerautomate.com
- CORS methods: GET, POST, PUT, PATCH, DELETE
```

![GitHub Copilot Chat &ndash; Global #1][image-06]

Then, it generates the following policy document.

![GitHub Copilot Chat &ndash; Global #2][image-07]

If you're happy with the result, click the "Insert at Cursor" menu to insert the policy document into the `policy-global.xml` file on the right.

![GitHub Copilot Chat &ndash; Global #3][image-08]

Then, you'll have the global-level policy document like below:

![GitHub Copilot Chat &ndash; Global #4][image-09]

Of course, you can open a new XML document and insert the result.

![GitHub Copilot Chat &ndash; Global #5][image-10]

As long as you're happy with that, that's fine. But in most cases, you might need to modify the policy document. Let's modify the policy document with GitHub Copilot. Add the following comment right after the `</allowed-methods>` tag and press the Enter key.

```xml
</allowed-methods>
<!-- add the allowed-headers node and accept everything -->
```

GitHub Copilot will suggest something like below. Press the Tab key to accept it.

![GitHub Copilot &ndash; Global #6][image-11]

Every time you hit the enter key, GitHub Copilot will suggest something you might want. Repeat this process and modify the policy document as follows:

![GitHub Copilot &ndash; Global #7][image-12]

Let's add response header policy in the global policy document. Add the following comment right after the `</allowed-headers>` tag and press the Enter key.

```xml
</allowed-headers>
<!-- add the expose-headers node and accept everything -->
```

Accept the suggestion from GitHub Copilot, if you're happy with that.

![GitHub Copilot &ndash; Global #8][image-13]

Keep repeating this until you get what you want.

![GitHub Copilot &ndash; Global #9][image-14]

Now you've got the global policy document. Save it as `policy-global.xml`.

### API-level policy document

Let's write an API-level policy document. Here's the scenario:

> Apply the same API key to all endpoints of the API. Assume that the API key is already stored in the APIM's Named Values feature.

Open Visual Studio Code and open the GitHub Copilot Chat window. Enter the following zero-shot prompt:

```yaml
Show me the Azure API Management policy document at the API level, including the following.

- Request header insertion
- Header name: x-functions-key
- Header value: API key value stored in the Named Values feature as "{{X_FUNCTIONS_KEY}}"
```

You might get something like below:

![GitHub Copilot &ndash; API #1][image-15]

If you want to store this policy document as a new file, you can do so by clicking the "Insert into a New File" menu.

![GitHub Copilot &ndash; API #2][image-16]

You have the new file.

![GitHub Copilot &ndash; API #3][image-17]

Save this file as `policy-api.xml`.

At the API level, GitHub Copilot has suggested the policy document that fulfills the scenario. If you need some more, you can open the `policy-api.xml` file and add more policies with GitHub Copilot like what you did for the global policy document.

### Operation-level policy document

Finally, let's write an operation-level policy document. Here's the scenario:

> For the `/products/{id}` operation, rewrite the URL to `/products?id={id}` and change the backend server address to `https://fabrikam.com/api`.

Within the GitHub Copilot Chat window, enter the following zero-shot prompt:

```yaml
Show me the Azure API Management policy document at the operation level, including the following.

- URL rewriting: Change /products/{id} to /products?id={id}
- Backend server URL: https://fabrikam.com/api
```

Here's the suggestion from GitHub Copilot:

![GitHub Copilot &ndash; Operation #1][image-18]

If you're happy with that, save it as `policy-operation.xml`, by clicking the "Insert into New File" menu. However, the policy document is not quite complete yet. You need to move the the `<set-backend-service>` node to either the `<inbound>` node or the `<backend>` node.

Once you've done, copy all those documents and paste them into the APIM portal.

---

So far, I've demonstrated how GitHub Copilot helps us write the APIM policy documents at the global, API and operation levels. As I mentioned at the beginning, writing or modifying the APIM policy documents can be cumbersome, and it has a fairly bit amount of learning curves. However, if you use GitHub Copilot, you can write the APIM policy documents much easier and faster.

## More about APIM...

If you want to learn more about APIM and APIM policies, the following links might be helpful.

- [Azure API Management key concepts][apim]
- [Implement API Management][apim learn]

[image-01]: /2023/07/gh-copilot-for-apim-policies-01.png
[image-02]: /2023/07/gh-copilot-for-apim-policies-02.png
[image-03]: /2023/07/gh-copilot-for-apim-policies-03.png
[image-04]: /2023/07/gh-copilot-for-apim-policies-04.png
[image-05]: /2023/07/gh-copilot-for-apim-policies-05.png
[image-06]: /2023/07/gh-copilot-for-apim-policies-06.png
[image-07]: /2023/07/gh-copilot-for-apim-policies-07.png
[image-08]: /2023/07/gh-copilot-for-apim-policies-08.png
[image-09]: /2023/07/gh-copilot-for-apim-policies-09.png
[image-10]: /2023/07/gh-copilot-for-apim-policies-10.png
[image-11]: /2023/07/gh-copilot-for-apim-policies-11.png
[image-12]: /2023/07/gh-copilot-for-apim-policies-12.png
[image-13]: /2023/07/gh-copilot-for-apim-policies-13.png
[image-14]: /2023/07/gh-copilot-for-apim-policies-14.png
[image-15]: /2023/07/gh-copilot-for-apim-policies-15.png
[image-16]: /2023/07/gh-copilot-for-apim-policies-16.png
[image-17]: /2023/07/gh-copilot-for-apim-policies-17.png
[image-18]: /2023/07/gh-copilot-for-apim-policies-18.png

[apim]: https://learn.microsoft.com/azure/api-management/api-management-key-concepts?WT.mc_id=dotnet-102583-juyoo
[apim policies]: https://learn.microsoft.com/azure/api-management/api-management-howto-policies?WT.mc_id=dotnet-102583-juyoo
[apim learn]: https://learn.microsoft.com/training/paths/az-204-implement-api-management/?WT.mc_id=dotnet-102583-juyoo

[aoai]: https://learn.microsoft.com/azure/ai-services/openai/overview?WT.mc_id=dotnet-102583-juyoo
[aoai prompt-engineering]: https://learn.microsoft.com/azure/ai-services/openai/concepts/prompt-engineering?WT.mc_id=dotnet-102583-juyoo#examples

[gh copilot]: https://docs.github.com/copilot/quickstart
[gh copilot subscription]: https://docs.github.com/billing/managing-billing-for-github-copilot/about-billing-for-github-copilot
[gh copilot chat beta]: https://github.blog/2023-07-20-github-copilot-chat-beta-now-available-for-every-organization

[gh student dev pack]: https://education.github.com/pack

[vs code]: https://code.visualstudio.com?WT.mc_id=dotnet-102583-juyoo
[vs code extensions copilot]: https://marketplace.visualstudio.com/items?itemName=GitHub.copilot&WT.mc_id=dotnet-102583-juyoo
[vs code extensions copilot chat]: https://marketplace.visualstudio.com/items?itemName=GitHub.copilot-chat&WT.mc_id=dotnet-102583-juyoo
