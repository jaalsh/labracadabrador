---
title: 'Observability via a correlation id in HTTP requests between .NET microservices'
date: 2023-05-14
# weight: 1
# aliases: ["/first"]
tags: ['microservices', '.net', 'c#', 'correlation-id', 'observability']
author: 'Jamie Sharpe'
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: true
description: 'Observability via a correlation id in HTTP requests between .NET microservices'
disableShare: false
disableHLJS: false
hideSummary: true
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: false
UseHugoToc: true
cover:
    image: '<image path/url>' # image path/url
    alt: '<alt text>' # alt text
    caption: '<text>' # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---

## Scenario

One issue with microservices and communication between them via HTTP is tracking the users request once it leaves the application and goes off to another application to do something else. For example, lets say we have 2 applications:

1. Ice cream order app
2. Ice cream stock management system

A user places an order for some ice cream in our ice cream order app, as part of the order request, the app makes a HTTP request to the ice cream stock management system to update the stock count.

What happens if an error occurs in the stock management system, how do we trace this back to the original request? In other words, how do track requests across application boundaries.

## Enter the correlation id

Using middleware we can attach a correlation id to HTTP requests as they come into the application. When used, the simple extension class below will attach a GUID correlation id to the headers of HTTP requests if one does not already exist.

```C#
public static class ObservabilityExtensions
{
    public const string CorrelationIdKey = "X-Correlation-Id";

    public static IApplicationBuilder UseCorrelationId(this IApplicationBuilder applicationBuilder)
    {
        applicationBuilder.Use(async (httpContext, next) =>
        {
            var correlationGuid = Guid.NewGuid();
            httpContext.Request.Headers.TryAdd(CorrelationIdKey, correlationGuid.ToString("N"));

            await next();
        });

        return applicationBuilder;
    }
}
```

We can then call `UseCorrelationId()` in our startup/program class.

```C#
 app.UseCorrelationId();
```

## Header propagation

So now each request will have a GUID correlation id with its headers, but how do we propagate the header around?

Thankfully .NET gives us a way of doing that, again using middleware.

First enable the header propagation middleware:

```C#
app.UseHeaderPropagation();
```

Then add the service and configure the options to propagate our correlation id header:

```C#
services.AddHeaderPropagation(options =>
    options.Headers.Add(ObservabilityExtensions.CorrelationIdKey)
);

```

Lastly, you need to update your Http Clients:

```C#
services.AddHttpClient("MyClientName")
    .AddHeaderPropagation();
```

Once done, the correlation id will be propagated from our Ice cream order app to the stock management system:

![Ice Cream Correlation Id Map Diagram](/ice_cream_correlation_id.png)

This means it can be logged as either a property or as part of the message, allowing you to track requests across application boundaries.

## Bonus

If you want an easy way to access the correlation id then you can update `ObservabilityExtensions` class to add a method to get the correlation id from the request headers.

```C#
public static class ObservabilityExtensions
{
    public const string CorrelationIdKey = "X-Correlation-Id";

    public static IApplicationBuilder UseCorrelationId(this IApplicationBuilder applicationBuilder)
    {
        applicationBuilder.Use(async (httpContext, next) =>
        {
            var correlationGuid = Guid.NewGuid();
            httpContext.Request.Headers.TryAdd(CorrelationIdKey, correlationGuid.ToString("N"));

            await next();
        });

        return applicationBuilder;
    }

    public static string GetCorrelationId(this HttpContext httpContext)
    {
        if (httpContext.Request.Headers.TryGetValue(CorrelationIdKey, out var correlationId))
        {
            return correlationId;
        }

        return string.Empty;
    }
}
```
