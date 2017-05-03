---
layout: post-light-feature
title: "Implementing a CDN in your ASP.NET MVC website"
description: "I hit quite a few stumbling blocks when trying to add a CDN to MaxTo.net. Learn from my mistakes here."
category: articles
tags: [microsoft, asp.net, c#, dotnet, cdn]
date: 2017-05-03
image: 
  feature: 2015-07-25/banner.png
---

Adding a CDN can greatly decrease the loading times of your website. The basic idea is that you store any static content on a server closer to the user, which means lower latency and higher download speeds.

What do you need to follow this short guide:

- An existing ASP.NET MVC website.
- An Azure account.

# Azure CDN

Azure offers 3 different CDN pricing tiers at this stage;

- Standard Akamai
- Standard Verizon
- Premium Verizon

Because of [a restriction in the ASP.NET bundling and optimization](https://feedback.azure.com/forums/169397-cdn/suggestions/6275266-cdn-support-vary-origin-header), we are going to need to go with the **Premium Verizon** option.

This option lets us use a "Rules engine", which you can use to change response headers before sending it back. We need this to send the correct CORS headers (`Access-Control-Allow-*`) when using the CDN to host CSS, Javascript and web fonts.

The problem with the Verizon range of CDNs, is that it takes a long time to provision (up to 1.5 hours) and changes to the rules takes an even longer time to deploy (4 hours). Thus it is important that you make sure you do it correctly the first time, too shorten your wait to a manageable 4 hours<sup id="a1">[1](#f1)</sup>.

## Setup

1. Search the Azure Marketplace for CDN, and create the CDN account. Make sure to select **Premium Verizon**.
2. Create an endpoint, which creates the domain name for your CDN. You can create multiple endpoints within one account.
   - The **Origin type** can be either **Custom domain** or **Web app**.
   - You can safely set the CDN to only reply via HTTPS (so remove the checkbox from HTTP)
   - Set the **Origin host header** to the same value as **Origin hostname**
3. Click the **Manage** link on the CDN account to open an external dashboard.
4. Under **HTTP LARGE**, find **Cache settings**, then **Query-string caching**.
5. Set the value to `unique-cache`.
6. Under **HTTP LARGE**, find **Rules engine**.
7. Follow the instructions from [this MSDN article](https://docs.microsoft.com/en-us/azure/cdn/cdn-cors). 
   - Pay close attention to the regular expression you use, as it determines all the valid `Origin` headers that will be allowed. 
   - If you will be hosting web fonts on this CDN, you also need to allow the CDN endpoint itself here. 
   - If you will be doing testing with on a local development machine, you should add your local development domain and port here.
   - This is not the place to enforce HTTPS (because in most local development you will not be using HTTPS). 
   - My regular expression: `https?:\/\/(www\.maxto\.net|maxto\.net|localhost:24207|maxto\.azureedge\.net)$`
8. Wait 4 hours.

# MVC

To actually use the CDN to get content, you will need to change a few URLs around. Most of these changes are just recommendations, and it could very well be that you may want to do some of these things differently in your project.

## Storing the CDN URL 

Open up `web.config` and add a setting for the CDN URL:

```xml
<configuration>
  <appSettings>
    <add key="CdnUrl" value="https://maxto.azureedge.net/" />
  </appSettings>
</configuration>
```

## Move bundling and minification to use CDN domain

ASP.NET MVC's bundling actually support CDNs "out of the box", but you need to make a few changes to make it work. First and foremost, it allows you to specify a CDN URL per bundle. This is great, but once you do that, the automatic cache breaker is not included. We will therefore need to recreate this.

I prefer to do this by creating a static value when the website starts, and keeping this value around until the web server is restarted. This ensures that every time I deploy an update to the website, I get a new cache breaker. This will not work if you have multiple server instances, however, in which case you could use the version number of your app (if you in fact version your web app).

So in my `BundleConfig.cs` file, I make a few changes:

```cs
public static void RegisterBundles(BundleCollection bundles)
{
    // runs once on startup, generating a new version each time
    var cacheBreaker = "?v=" + Convert.ToBase64String(Guid.NewGuid().ToByteArray());
    var cdnUrl = ConfigurationManager.AppSettings["CdnUrl"];
    
    // for each bundle, add the second parameter here:
    var lessCoreBundle = new Bundle(
        "~/bundles/core.css", 
        cdnUrl + "bundles/core.css" + cacheBreaker);
    // the rest of your configuration goes here
    
    #if !DEBUG
    // enable the CDN when running in Release mode.
    bundles.UseCdn = true;
    #endif
}
```

Now start your app in Release mode, and your bundles should be requested from your CDN.

## Requesting images and other static content from CDN

You will need to change the links to any images and other static content you want to load from the CDN. If you've followed ASP.NET guidance, most of these links will be generated by `@Url.Content('~/foo.png')`, and we'll provide a simple extension method that will only require you to change these links to `@Url.CdnContent('~/foo.png')`.

Create this `UrlHelperExtensions.cs` file somewhere that makes sense for your project:

```cs
public static class UrlHelperExtensions
{
    // runs once on startup, generating a new version each time
    private static readonly string CacheBreaker = "?v=" + Convert.ToBase64String(Guid.NewGuid().ToByteArray());

    public static string CdnContent(this UrlHelper helper, string contentPath)
    {
#if !DEBUG
        // use CDN in Release mode
        var cdnUrl = ConfigurationManager.AppSettings["CdnUrl"];
        return cdnUrl.TrimEnd('/') + "/" + contentPath.TrimStart('~', '/') + CacheBreaker;
#else
        // use local URLs in Debug mode
        return helper.Content(contentPath);
#endif
    }
}
```

In the `Views/web.config` file, add the namespace of `UrlHelperExtensions`:

```xml
<configuration>
  <system.web.webPages.razor>
    <pages pageBaseType="System.Web.Mvc.WebViewPage">
      <namespaces>
        <add namespace="System.Web.Mvc" />
        <add namespace="System.Web.Mvc.Ajax" />
        <add namespace="System.Web.Mvc.Html" />
        <add namespace="System.Web.Optimization"/>
        <add namespace="System.Web.Routing" />
        <add namespace="MaxTo.Website.Infrastructure.Extensions" /> <!-- your namespace goes here -->
      </namespaces>
    </pages>
  </system.web.webPages.razor>
</configuration>
```

Now use Find all and search for `@Url.Content` and replace all the relevant URLs.

<b id="f1">1</b> I did, in fact, not do it correctly the first time. Nor the second. Or the third. [↩](#a1)
