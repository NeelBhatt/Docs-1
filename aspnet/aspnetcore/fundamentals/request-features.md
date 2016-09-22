---
uid: fundamentals/request-features
---
# Request Features

By [Steve Smith](http://ardalis.com)

Individual web server features related to how HTTP requests and responses are handled have been factored into separate interfaces. These abstractions are used by individual server implementations and middleware to create and modify the application's hosting pipeline.

## Feature interfaces

ASP.NET Core defines a number of HTTP feature interfaces in [Microsoft.AspNetCore.Http.Features](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/index.html.md#Microsoft.AspNetCore.Http.Features.md) which are used by servers to identify the features they support. The following feature interfaces handle requests and return responses:

[IHttpRequestFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/IHttpRequestFeature/index.html.md#Microsoft.AspNetCore.Http.Features.IHttpRequestFeature.md)
   Defines the structure of an HTTP request, including the protocol, path, query string, headers, and body.

[IHttpResponseFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/IHttpResponseFeature/index.html.md#Microsoft.AspNetCore.Http.Features.IHttpResponseFeature.md)
   Defines the structure of an HTTP response, including the status code, headers, and body of the response.

[IHttpAuthenticationFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/Authentication/IHttpAuthenticationFeature/index.html.md#Microsoft.AspNetCore.Http.Features.Authentication.IHttpAuthenticationFeature.md)
   Defines support for identifying users based on a `ClaimsPrincipal` and specifying an authentication handler.

[IHttpUpgradeFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/IHttpUpgradeFeature/index.html.md#Microsoft.AspNetCore.Http.Features.IHttpUpgradeFeature.md)
   Defines support for [HTTP Upgrades](https://tools.ietf.org/html/rfc2616.html.md#section-14.42.md), which allow the client to specify which additional protocols it would like to use if the server wishes to switch protocols.

[IHttpBufferingFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/IHttpBufferingFeature/index.html.md#Microsoft.AspNetCore.Http.Features.IHttpBufferingFeature.md)
   Defines methods for disabling buffering of requests and/or responses.

[IHttpConnectionFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/IHttpConnectionFeature/index.html.md#Microsoft.AspNetCore.Http.Features.IHttpConnectionFeature.md)
   Defines properties for local and remote addresses and ports.

[IHttpRequestLifetimeFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/IHttpRequestLifetimeFeature/index.html.md#Microsoft.AspNetCore.Http.Features.IHttpRequestLifetimeFeature.md)
   Defines support for aborting connections, or detecting if a request has been terminated prematurely, such as by a client disconnect.

[IHttpSendFileFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/IHttpSendFileFeature/index.html.md#Microsoft.AspNetCore.Http.Features.IHttpSendFileFeature.md)
   Defines a method for sending files asynchronously.

[IHttpWebSocketFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/IHttpWebSocketFeature/index.html.md#Microsoft.AspNetCore.Http.Features.IHttpWebSocketFeature.md)
   Defines an API for supporting web sockets.

[IHttpRequestIdentifierFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/IHttpRequestIdentifierFeature/index.html.md#Microsoft.AspNetCore.Http.Features.IHttpRequestIdentifierFeature.md)
   Adds a property that can be implemented to uniquely identify requests.

[ISessionFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/ISessionFeature/index.html.md#Microsoft.AspNetCore.Http.Features.ISessionFeature.md)
   Defines `ISessionFactory` and `ISession` abstractions for supporting user sessions.

[ITlsConnectionFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/ITlsConnectionFeature/index.html.md#Microsoft.AspNetCore.Http.Features.ITlsConnectionFeature.md)
   Defines an API for retrieving client certificates.

[ITlsTokenBindingFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/ITlsTokenBindingFeature/index.html.md#Microsoft.AspNetCore.Http.Features.ITlsTokenBindingFeature.md)
   Defines methods for working with TLS token binding parameters.

> [!NOTE]
> [ISessionFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/ISessionFeature/index.html.md#Microsoft.AspNetCore.Http.Features.ISessionFeature.md) is not a server feature, but is implemented by the [SessionMiddleware](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Session/SessionMiddleware/index.html.md#Microsoft.AspNetCore.Session.SessionMiddleware.md) (see [Managing Application State](app-state.md)).

## Feature collections

The [Features](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/HttpContext/index.html.md#Microsoft.AspNetCore.Http.HttpContext.Features.md) property of [HttpContext](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/HttpContext/index.html.md#Microsoft.AspNetCore.Http.HttpContext.md) provides an interface for getting and setting the available HTTP features for the current request. Since the feature collection is mutable even within the context of a request, middleware can be used to modify the collection and add support for additional features.

## Middleware and request features

While servers are responsible for creating the feature collection, middleware can both add to this collection and consume features from the collection. For example, the [StaticFileMiddleware](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/StaticFiles/StaticFileMiddleware/index.html.md#Microsoft.AspNetCore.StaticFiles.StaticFileMiddleware.md) accesses the [IHttpSendFileFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/IHttpSendFileFeature/index.html.md#Microsoft.AspNetCore.Http.Features.IHttpSendFileFeature.md) feature. If the feature exists, it is used to send the requested static file from its physical path. Otherwise, a slower alternative method is used to send the file. When available, the [IHttpSendFileFeature](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/IHttpSendFileFeature/index.html.md#Microsoft.AspNetCore.Http.Features.IHttpSendFileFeature.md) allows the operating
system to open the file and perform a direct kernel mode copy to the network card.

Additionally, middleware can add to the feature collection established by the server. Existing features can even be replaced by middleware, allowing the middleware to augment the functionality of the server. Features added to the collection are available immediately to other middleware or the underlying application itself later in the request pipeline.

By combining custom server implementations and specific middleware enhancements, the precise set of features an application requires can be constructed. This allows missing features to be added without requiring a change in server, and ensures only the minimal amount of features are exposed, thus limiting attack surface area and improving performance.

## Summary

Feature interfaces define specific HTTP features that a given request may support. Servers define collections of features, and the initial set of features supported by that server, but middleware can be used to enhance these features.

## Additional Resources

* [Servers](servers.md)

* [Middleware](middleware.md)

* [Open Web Interface for .NET (OWIN)](owin.md)
