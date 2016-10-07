---
uid: fundamentals/servers
---
>[!WARNING]
> This page documents version 1.0.0-rc1 and has not yet been updated for version 1.0.0

# Servers

By [Steve Smith](http://ardalis.com)

ASP.NET Core is completely decoupled from the web server environment that hosts the application. ASP.NET Core supports hosting in IIS and IIS Express, and self-hosting scenarios using the Kestrel and WebListener HTTP servers. Additionally, developers and third party software vendors can create custom servers to host their ASP.NET Core apps.

[View or download sample code](https://github.com/aspnet/Docs/tree/master/aspnet/fundamentals/servers/sample)

## Servers and commands

ASP.NET Core was designed to decouple web applications from the underlying HTTP server. Traditionally, ASP.NET apps have been windows-only hosted on Internet Information Server (IIS). The recommended way to run ASP.NET Core applications on Windows is using IIS as a reverse-proxy server. The HttpPlatformHandler module in IIS manages and proxies requests to an HTTP server hosted out-of-process. ASP.NET Core ships with two different HTTP servers:

* Microsoft.AspNetCore.Server.Kestrel (AKA Kestrel, cross-platform)

* Microsoft.AspNetCore.Server.WebListener (AKA WebListener, Windows-only, preview)

ASP.NET Core does not directly listen for requests, but instead relies on the HTTP server implementation to surface the request to the application as a set of [feature interfaces](request-features.md) composed into an HttpContext. While WebListener is Windows-only, Kestrel is designed to run cross-platform. You can configure your application to be hosted by any or all of these servers by specifying commands in your *project.json* file. You can even specify an application entry point for your application, and run it as an executable (using `dotnet run`) rather than hosting it in a separate process.

The default web host for ASP.NET apps developed using Visual Studio is IIS Express functioning as a reverse proxy server for Kestrel. The "Microsoft.AspNetCore.Server.Kestrel" and "Microsoft.AspNetCore.Server.IISIntegration" dependencies are included in *project.json* by default, even with the Empty web site template. Visual Studio provides support for multiple profiles, associated with IIS Express. You can manage these profiles and their settings in the **Debug** tab of your web application project's Properties menu or from the *launchSettings.json* file.

![image](servers/_static/serverdemo-properties.png)

The sample project for this article is configured to support each server option in the *project.json* file:

project.json (truncated)

[!code-json[Main](../fundamentals/servers/sample/ServersDemo/src/ServersDemo/project.json?highlight=12,13)]

````json

   {
     "webroot": "wwwroot",
     "version": "1.0.0-*",

     "dependencies": {
       "Microsoft.AspNet.Server.Kestrel": "1.0.0-rc1-final",
       "Microsoft.AspNet.Server.WebListener": "1.0.0-rc1-final"
     },

     "commands": {
       "run": "run server.urls=http://localhost:5003",
       "web": "Microsoft.AspNet.Hosting --server Microsoft.AspNet.Server.Kestrel --server.urls http://localhost:5000",
       "weblistener": "Microsoft.AspNet.Hosting --server WebListener --server.urls http://localhost:5004"
     },

     "frameworks": {
       "dnx451": { },

   ````

The `run` command will launch the application from the `void main` method. The `run` command configures and starts an instance of `Kestrel`.

program.cs

[!code-csharp[Main](../fundamentals/servers/sample/ServersDemo/src/ServersDemo/Program.cs?highlight=32,33,34,35,36,37,38,39,40)]

````csharp

   using System;
   using System.Threading.Tasks;
   using Microsoft.AspNet.Hosting;
   using Microsoft.Extensions.Configuration;
   using Microsoft.AspNet.Builder;
   using Microsoft.Extensions.Logging;
   using Microsoft.AspNet.Server.Kestrel;

   namespace ServersDemo
   {
       /// <summary>
       /// This demonstrates how the application can be launched in a console application. 
       /// Executing the "dnx run" command in the application folder will run this app.
       /// </summary>
       public class Program
       {
           private readonly IServiceProvider _serviceProvider;

           public Program(IServiceProvider serviceProvider)
           {
               _serviceProvider = serviceProvider;
           }

           public Task<int> Main(string[] args)
           {
               //Add command line configuration source to read command line parameters.
               var builder = new ConfigurationBuilder();
               builder.AddCommandLine(args);
               var config = builder.Build();

               using (new WebHostBuilder(config)
                   .UseServer("Microsoft.AspNet.Server.Kestrel")
                   .Build()
                   .Start())
               {
                   Console.WriteLine("Started the server..");
                   Console.WriteLine("Press any key to stop the server");
                   Console.ReadLine();
               }
               return Task.FromResult(0);
           }
       }
   }

   ````

## Supported Features by Server

ASP.NET defines a number of [Request Features](request-features.md). The following table lists the WebListener and Kestrel support for request features.

<!--       Feature  WebListener  Kestrel  IHttpRequestFeature  Yes  Yes  IHttpResponseFeature  Yes  Yes  IHttpAuthenticationFeature  Yes  No  IHttpUpgradeFeature  Yes (with limits)  Yes  IHttpBufferingFeature  Yes  No  IHttpConnectionFeature  Yes  Yes  IHttpRequestLifetimeFeature  Yes  Yes  IHttpSendFileFeature  Yes  No  IHttpWebSocketFeature  No*  No*  IRequestIdentifierFeature  Yes  No  ITlsConnectionFeature  Yes  Yes  ITlsTokenBindingFeature  Yes  No -->  ### Configuration options

You can provide configuration options (by command line parameters or a configuration file) that are read on server startup.

The `Microsoft.AspNetCore.Hosting` command supports server parameters (such as `Kestrel` or `WebListener`) and a `server.urls` configuration key. The `server.urls` configuration key is a semicolon-separated list of URL prefixes that the server should handle.

The *project.json* file shown above demonstrates how to pass the `server.urls` parameter directly:

````javascript

   "web": "Microsoft.AspNetCore.Kestrel --server.urls http://localhost:5004"
   ````

Alternately, a  JSON configuration file can be used,

````javascript

   "kestrel": "Microsoft.AspNetCore.Hosting"
   ````

The `hosting.json` can include the settings the server will use (including the server parameter, as well):

````json

   {
     "server": "Microsoft.AspNetCore.Server.Kestrel",
     "server.urls": "http://localhost:5004/"
   }
   ````

### Programmatic configuration

The server hosting the application can be referenced programmatically via the [`IApplicationBuilder`](https://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Builder/IApplicationBuilder/index.html) interface, available in the `Configure` method in `Startup`. [`IApplicationBuilder`](https://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Builder/IApplicationBuilder/index.html) exposes Server Features of type [`IFeatureCollection`](https://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/IFeatureCollection/index.html). `IServerAddressesFeature` only expose a `Addresses` property, but different server implementations may expose additional functionality. For instance, WebListener exposes `AuthenticationManager` that can be used to configure the server's authentication:

[!code-csharp[Main](../fundamentals/servers/sample/ServersDemo/src/ServersDemo/Startup.cs?highlight=3,6,7,10,15)]

````csharp

   public void Configure(IApplicationBuilder app, IApplicationLifetime lifetime, ILoggerFactory loggerFactory)
   {
       var webListenerInfo = app.ServerFeatures.Get<WebListener>();
       if (webListenerInfo != null)
       {
           webListenerInfo.AuthenticationManager.AuthenticationSchemes =
               AuthenticationSchemes.AllowAnonymous;
       }

       var serverAddress = app.ServerFeatures.Get<IServerAddressesFeature>()?.Addresses.FirstOrDefault();

       app.Run(async (context) =>
       {
           var message = String.Format("Hello World from {0}",
                                   serverAddress);
           await context.Response.WriteAsync(message);
       });
   }

   ````

## IIS and IIS Express

IIS is the most feature rich server, and includes IIS management functionality and access to other IIS modules. Hosting ASP.NET Core no longer uses the `System.Web` infrastructure used by prior versions of ASP.NET.

### ASP.NET Core Module

In ASP.NET Core on Windows, the web application is hosted by an external process outside of IIS. The ASP.NET Core Module is a native IIS  module that is used to proxy requests to external processes that it manages. See [ASP.NET Core Module Configuration Reference](../hosting/aspnet-core-module.md) for more details.

<a name=weblistener></a>

## WebListener

WebListener is a Windows-only HTTP server for ASP.NET Core. It runs directly on the [Http.Sys kernel driver](http://www.iis.net/learn/get-started/introduction-to-iis/introduction-to-iis-architecture), and has very little overhead.

You can add support for WebListener to your ASP.NET application by adding the "Microsoft.AspNetCore.Server.WebListener" dependency in *project.json* and the following command:

````javascript

   "web": "Microsoft.AspNetCore.Hosting --server Microsoft.AspNetCore.Server.WebListener --server.urls http://localhost:5000"
   ````

> [!NOTE]
> WebListener is currently still in preview.

<a name=kestrel></a>

## Kestrel

Kestrel is a cross-platform web server based on [libuv](https://github.com/libuv/libuv), a cross-platform asynchronous I/O library. You add support for Kestrel by including `Microsoft.AspNetCore.Server.Kestrel` in your project's dependencies listed in *project.json*.

Learn more about working with Kestrel to create [Your First ASP.NET Core Application on a Mac Using Visual Studio Code](../tutorials/your-first-mac-aspnet.md).

> [!NOTE]
> Kestrel is designed to be run behind a proxy (for example IIS or Nginx) and should not be deployed directly facing the Internet.

## Choosing a server

If you intend to deploy your application on a Windows server, you should run IIS as a reverse proxy server that manages and proxies requests to Kestrel. If deploying on Linux, you should run a comparable reverse proxy server such as Apache or Nginx to proxy requests to Kestrel (see [Publish to a Linux Production Environment](../publishing/linuxproduction.md)).

## Custom Servers

You can create your own server in which to host ASP.NET apps, or use other open source servers. When implementing your own server, you're free to implement just the feature interfaces your application needs, though at a minimum you must support [`IHttpRequestFeature`](https://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/IHttpRequestFeature/index.html) and [`IHttpResponseFeature`](https://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/Features/IHttpResponseFeature/index.html).

Since Kestrel is open source, it makes an excellent starting point if you need to implement your own custom server. Like all of ASP.NET Core, you're welcome to [contribute](https://github.com/aspnet/KestrelHttpServer/blob/dev/CONTRIBUTING.md) any improvements you make back to the project.

Kestrel currently supports a limited number of feature interfaces, but additional features will be added in the future.

## Additional Reading

* [Request Features](request-features.md)
