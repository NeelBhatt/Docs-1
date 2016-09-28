---
uid: migration/http-modules
---
>[!WARNING]
> This page documents version 1.0.0-rc1 and has not yet been updated for version 1.0.0

# Migrating HTTP Modules to Middleware

By [Matt Perdeck](http://www.linkedin.com/in/mattperdeck)

This article shows how to migrate existing ASP.NET [HTTP modules and handlers](https://msdn.microsoft.com/en-us/library/bb398986.aspx) to ASP.NET Core [middleware](../fundamentals/middleware.md#fundamentals-middleware.md).

## Handlers and modules revisited

Before proceeding to ASP.NET Core middleware, let's first recap how HTTP modules and handlers work:

![image](http-modules/_static/moduleshandlers.png)

Handlers are:
   * Classes that implement [IHttpHandler](https://msdn.microsoft.com/en-us/library/system.web.ihttphandler(v=vs.100).aspx)

   * Used to handle requests with a given file name or extension, such as *.report*

   * [Configured](https://msdn.microsoft.com/en-us/library/46c5ddfy(v=vs.100).aspx) in *Web.config*

Modules are:
   * Classes that implement [IHttpModule](https://msdn.microsoft.com/en-us/library/system.web.ihttpmodule(v=vs.100).aspx)

   * Invoked for every request

   * Able to short-circuit (stop further processing of a request)

   * Able to add to the HTTP response, or create their own

   * [Configured](https://msdn.microsoft.com/en-us/library/ms227673(v=vs.100).aspx) in *Web.config*

**The order in which modules process incoming requests is determined by:**

   1. The [application life cycle](https://msdn.microsoft.com/en-us/library/ms227673(v=vs.100).aspx), which is a series events fired by ASP.NET: [BeginRequest](https://msdn.microsoft.com/en-us/library/system.web.httpapplication.beginrequest(v=vs.100).aspx), [AuthenticateRequest](https://msdn.microsoft.com/en-us/library/system.web.httpapplication.authenticaterequest(v=vs.100).aspx), etc. Each module can create a handler for one or more events.

   2. For the same event, the order in which they are configured in *Web.config*.

In addition to modules, you can add handlers for the life cycle events to your *Global.asax.cs* file. These handlers run after the handlers in the configured modules.

## From handlers and modules to middleware

Middleware are simpler than HTTP modules and handlers:
   * Modules, handlers, *Global.asax.cs*, *Web.config* (except for IIS configuration) and the application life cycle are gone

   * The roles of both modules and handlers have been taken over by middleware

   * Middleware are configured using code rather than in *Web.config*

   * [Pipeline branching](../fundamentals/middleware.md#middleware-run-map-use.md) lets you send requests to specific middleware, based on not only the URL but also on request headers, query strings, etc.

Middleware are very similar to modules:
   * Invoked in principle for every request

   * Able to short-circuit a request, by [not passing the request to the next middleware](xref:migration/http-modules#http-modules-shortcircuiting-middleware)

   * Able to create their own HTTP response

Middleware and modules are processed in a different order:
   * Order of middleware is based on the order in which they are inserted into the request pipeline, while order of modules is mainly based on [application life cycle](https://msdn.microsoft.com/en-us/library/ms227673(v=vs.100).aspx) events

   * Order of middleware for responses is the reverse from that for requests, while order of modules is the same for requests and responses

   * See [Creating a middleware pipeline with IApplicationBuilder](../fundamentals/middleware.html#creating-a-middleware-pipeline-with-iapplicationbuilder)

![image](http-modules/_static/middleware.png)

Note how in the image above, the authentication middleware short-circuited the request.

## Migrating module code to middleware

An existing HTTP module will look similar to this:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net4/Asp.Net4/Modules/MyModule.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [6, 8, 24, 31], "linenostart": 1}} -->

````c#

   // ASP.NET 4 module

   using System;
   using System.Web;

   namespace MyApp.Modules
   {
       public class MyModule : IHttpModule
       {
           public void Dispose()
           {
           }

           public void Init(HttpApplication application)
           {
               application.BeginRequest += (new EventHandler(this.Application_BeginRequest));
               application.EndRequest += (new EventHandler(this.Application_EndRequest));
           }

           private void Application_BeginRequest(Object source, EventArgs e)
           {
               HttpContext context = ((HttpApplication)source).Context;

               // Do something with context near the beginning of request processing.
           }

           private void Application_EndRequest(Object source, EventArgs e)
           {
               HttpContext context = ((HttpApplication)source).Context;

               // Do something with context near the end of request processing.
           }
       }
   }

   ````

As shown in the [Middleware](../fundamentals/middleware.md) page, an ASP.NET Core middleware is simply a class that exposes an `Invoke` method taking an `HttpContext` and returning a `Task`. Your new middleware will look like this:

<a name=http-modules-usemiddleware></a>

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/MyMiddleware.cs", "ids": ["http-modules-usemiddleware"], "dupnames": [], "names": ["http-modules-usemiddleware"], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [9, 13, 20, 24, 28, 30, 32], "linenostart": 1}} -->

````c#

   // ASP.NET 5 middleware

   using Microsoft.AspNet.Builder;
   using Microsoft.AspNet.Http;
   using System.Threading.Tasks;

   namespace MyApp.Middleware
   {
       public class MyMiddleware
       {
           private readonly RequestDelegate _next;

           public MyMiddleware(RequestDelegate next)
           {
               _next = next;
           }

           public async Task Invoke(HttpContext context)
           {
               // Do something with context near the beginning of request processing.

               await _next.Invoke(context);

               // Clean up.
           }
       }

       public static class MyMiddlewareExtensions
       {
           public static IApplicationBuilder UseMyMiddleware(this IApplicationBuilder builder)
           {
               return builder.UseMiddleware<MyMiddleware>();
           }
       }
   }

   ````

The above middleware template was taken from the section on [writing middleware](../fundamentals/middleware.md#middleware-writing-middleware.md).

The *MyMiddlewareExtensions* helper class makes it easier to configure your middleware in your `Startup` class. The `UseMyMiddleware` method adds your middleware class to the request pipeline. Services required by the middleware get injected in the middleware's constructor.

<a name=http-modules-shortcircuiting-middleware></a>

Your module might terminate a request, for example if the user is not authorized:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net4/Asp.Net4/Modules/MyTerminatingModule.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [9, 10, 11, 12, 13], "linenostart": 1}} -->

````c#

   // ASP.NET 4 module that may terminate the request

   private void Application_BeginRequest(Object source, EventArgs e)
   {
       HttpContext context = ((HttpApplication)source).Context;

       // Do something with context near the beginning of request processing.

       if (TerminateRequest())
       {
           context.Response.End();
           return;
       }
   }

   ````

A middleware handles this by simply not calling `Invoke` on the next middleware in the pipeline. Keep in mind that this does not fully terminate the request, because previous middlewares will still be invoked when the response makes its way back through the pipeline.

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/MyTerminatingMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [7, 8], "linenostart": 1}} -->

````c#

   // ASP.NET 5 middleware that may terminate the request

   public async Task Invoke(HttpContext context)
   {
       // Do something with context near the beginning of request processing.

       if (!TerminateRequest())
           await _next.Invoke(context);

       // Clean up.
   }

   ````

When you migrate your module's functionality to your new middleware, you may find that your code doesn't compile because the `HttpContext` class has significantly changed in ASP.NET Core. [Later on](#migrating-to-the-new-httpcontext), you'll see how to migrate to the new ASP.NET Core HttpContext.

## Migrating module insertion into the request pipeline

HTTP modules are typically added to the request pipeline using *Web.config*:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net4/Asp.Net4/Web.config", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "xml", "highlight_args": {"hl_lines": [6], "linenostart": 1}} -->

````xml

   <?xml version="1.0" encoding="utf-8"?>
   <!--ASP.NET 4 web.config-->
   <configuration>
     <system.webServer>
       <modules>
         <add name="MyModule" type="MyApp.Modules.MyModule"/>
       </modules>
     </system.webServer>
   </configuration>
   ````

Convert this by [adding your new middleware](../fundamentals/middleware.html#creating-a-middleware-pipeline-with-iapplicationbuilder) to the request pipeline in your `Startup` class:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Startup.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [12], "linenostart": 1}} -->

````c#

   // ASP.NET 5 Startup class

   namespace Asp.Net5
   {
       public class Startup
       {
           public void Configure(IApplicationBuilder app, IHostingEnvironment env, 
               ILoggerFactory loggerFactory)
           {
               // ...

               app.UseMyMiddleware();

               // ...
           }
       }
   }

   ````

The exact spot in the pipeline where you insert your new middleware depends on the event that it handled as a module (`BeginRequest`, `EndRequest`, etc.) and its order in your list of modules in *Web.config*.

As previously stated, there is no more application life cycle in ASP.NET Core and the order in which responses are processed by middleware differs from the order used by modules. This could make your ordering decision more  challenging.

If ordering becomes a problem, you could split your module into multiple middleware that can be ordered independently.

## Migrating handler code to middleware

An HTTP handler looks something like this:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net4/Asp.Net4/HttpHandlers/ReportHandler.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [5, 7, 13, 14, 15, 16], "linenostart": 1}} -->

````c#

   // ASP.NET 4 handler

   using System.Web;

   namespace MyApp.HttpHandlers
   {
       public class MyHandler : IHttpHandler
       {
           public bool IsReusable { get { return true; } }

           public void ProcessRequest(HttpContext context)
           {
               string response = GenerateResponse(context);

               context.Response.ContentType = GetContentType();
               context.Response.Output.Write(response);
           }

           // ...
       }
   }

   ````

In your ASP.NET Core project, you would translate this to a middleware similar to this:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/ReportHandlerMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [7, 9, 13, 20, 21, 22, 23, 29, 31, 33], "linenostart": 1}} -->

````c#

   // ASP.NET 5 middleware migrated from a handler

   using Microsoft.AspNet.Builder;
   using Microsoft.AspNet.Http;
   using System.Threading.Tasks;

   namespace MyApp.Middleware
   {
       public class MyHandlerMiddleware
       {

           // Must have constructor with this signature, otherwise exception at run time
           public MyHandlerMiddleware(RequestDelegate next)
           {
               // This is an HTTP Handler, so no need to store next
           }

           public async Task Invoke(HttpContext context)
           {
               string response = GenerateResponse(context);

               context.Response.ContentType = GetContentType();
               await context.Response.WriteAsync(response);
           }

           // ...
       }

       public static class MyHandlerExtensions
       {
           public static IApplicationBuilder UseMyHandler(this IApplicationBuilder builder)
           {
               return builder.UseMiddleware<MyHandlerMiddleware>();
           }
       }
   }

   ````

This middleware is very similar to the middleware corresponding to modules. The only real difference is that here there is no call to `_next.Invoke(context)`. That makes sense, because the handler is at the end of the request pipeline, so there will be no next middleware to invoke.

## Migrating handler insertion into the request pipeline

Configuring an HTTP handler is done in *Web.config* and looks something like this:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net4/Asp.Net4/Web.config", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "xml", "highlight_args": {"hl_lines": [6], "linenostart": 1}} -->

````xml

   <?xml version="1.0" encoding="utf-8"?>
   <!--ASP.NET 4 web.config-->
   <configuration>
     <system.webServer>
       <handlers>
         <add name="MyHandler" verb="*" path="*.report" type="MyApp.HttpHandlers.MyHandler" resourceType="Unspecified" preCondition="integratedMode"/>
       </handlers>
     </system.webServer>
   </configuration>
   ````

You could convert this by adding your new handler middleware to the request pipeline in your `Startup` class, similar to middleware converted from modules. The problem with that approach is that it would send all requests to your new handler middleware. However, you only want requests with a given extension to reach your middleware. That would give you the same functionality you had with your HTTP handler.

One solution is to branch the pipeline for requests with a given extension, using the `MapWhen` extension method. You do this in the same `Configure` method where you add the other middleware:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Startup.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [12, 13, 14, 15, 16, 17], "linenostart": 1}} -->

````c#

   // ASP.NET 5 Startup class

   namespace Asp.Net5
   {
       public class Startup
       {
           public void Configure(IApplicationBuilder app, IHostingEnvironment env, 
               ILoggerFactory loggerFactory)
           {
               // ...

               app.MapWhen(
                   context => context.Request.Path.ToString().EndsWith(".report"),
                   appBranch => {
                       // ... optionally add more middleware to this branch
                       appBranch.UseMyHandler();
                   });
           }
       }
   }

   ````

`MapWhen` takes these parameters:

1. A lambda that takes the `HttpContext` and returns `true` if the request should go down the branch. This means you can branch requests not just based on their extension, but also on request headers, query string parameters, etc.

2. A lambda that takes an `IApplicationBuilder` and adds all the middleware for the branch. This means you can add additional middleware to the branch in front of your handler middleware.

Middleware added to the pipeline before the branch will be invoked on all requests; the branch will have no impact on them.

## Loading middleware options using the options pattern

Some modules and handlers have configuration options that are stored in *Web.config*. However, in ASP.NET Core a new configuration model is used in place of *Web.config*.

The new [configuration system](../fundamentals/configuration.md) gives you these options to solve this:

* Directly inject the options into the middleware, as shown in the [next section](xref:migration/http-modules#loading-middleware-options-through-direct-injection).

* Use the [options pattern](../fundamentals/configuration.md#options-config-objects.md):

1. Create a class to hold your middleware options, for example:

      <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/MyMiddlewareWithParams.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"linenostart": 1}} -->

      ````c#

         public class MyMiddlewareOptions
         {
             public string Param1 { get; set; }
             public string Param2 { get; set; }
         }

         ````

2. Store the option values

      The new configuration system allows you to essentially store option values anywhere you want. However, most sites use *appsettings.json*, so we'll take that approach:

      <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/appsettings.json", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "json", "highlight_args": {"linenostart": 1}} -->

      ````json

         {
           "MyMiddlewareOptionsSection": {
             "Param1": "Param1Value",
             "Param2": "Param2Value"
           }
         }

         ````

      *MyMiddlewareOptionsSection* here is simply a section name. It doesn't have to be the same as the name of your options class.

3. Associate the option values with the options class

      The options pattern uses ASP.NET Core's dependency injection framework to associate the options type (such as `MyMiddlewareOptions`) with an `MyMiddlewareOptions` object that has the actual options.

      Update your `Startup` class:

         1. If you're using *appsettings.json*, add it to the configuration builder in the `Startup` constructor:

         <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Startup.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [7], "linenostart": 1}} -->

         ````c#

            public class Startup
            {
                public Startup(IHostingEnvironment env)
                {
                    // Set up configuration sources.
                    var builder = new ConfigurationBuilder()
                        .AddJsonFile("appsettings.json")
                        .AddEnvironmentVariables();
                    Configuration = builder.Build();
                }

            }

            ````

         2. Configure the options service:

         <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Startup.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [5], "linenostart": 1}} -->

         ````c#

            public class Startup
            {
                public void ConfigureServices(IServiceCollection services)
                {
                    services.AddOptions();

                    // ...
                }
            }

            ````

         3. Associate your options with your options class:

         <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Startup.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [7, 8], "linenostart": 1}} -->

         ````c#

            public class Startup
            {
                public void ConfigureServices(IServiceCollection services)
                {
                    services.AddOptions();

                    services.Configure<MyMiddlewareOptions>(
                        Configuration.GetSection("MyMiddlewareOptionsSection"));

                    // ...
                }
            }

            ````

4. Inject the options into your middleware constructor. This is similar to injecting options into a controller.

      <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/MyMiddlewareWithParams.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [7, 10, 13, 19, 24], "linenostart": 1}} -->

      ````c#

         namespace MyApp.Middleware
         {

             public class MyMiddlewareWithParams
             {
                 private readonly RequestDelegate _next;
                 private readonly MyMiddlewareOptions _myMiddlewareOptions;

                 public MyMiddlewareWithParams(RequestDelegate next,
                     IOptions<MyMiddlewareOptions> optionsAccessor)
                 {
                     _next = next;
                     _myMiddlewareOptions = optionsAccessor.Value;
                 }

                 public async Task Invoke(HttpContext context)
                 {
                     // Do something with context near the beginning of request processing
                     // using configuration in _myMiddlewareOptions

                     await _next.Invoke(context);

                     // Do something with context near the end of request processing
                     // using configuration in _myMiddlewareOptions
                 }
             }

         ````

      The [UseMiddleware](#http-modules-usemiddleware) extension method that adds your middleware to the `IApplicationBuilder` takes care of dependency injection.

      This is not limited to `IOptions` objects. Any other object that your middleware requires can be injected this way.

<a name=loading-middleware-options-through-direct-injection></a>

## Loading middleware options through direct injection

The options pattern has the advantage that it creates loose coupling between options values and their consumers. Once you've associated an options class with the actual options values, any other class can get access to the options through the dependency injection framework. There is no need to pass around options values.

This breaks down though if you want to use the same middleware twice, with different options. For example an authorization middleware used in different branches allowing different roles. You can't associate two different options objects with the one options class.

The solution is to get the options objects with the actual options values in your `Startup` class and pass those directly to each instance of your middleware.

1. Add a second key to *appsettings.json*

      To add a second set of options to the *appsettings.json* file, simply use a new key to uniquely identify it:

      <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/appsettings.json", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "json", "highlight_args": {"hl_lines": [2, 3, 4, 5], "linenostart": 1}} -->

      ````json

         {
           "MyMiddlewareOptionsSection2": {
             "Param1": "Param1Value2",
             "Param2": "Param2Value2"
           },
           "MyMiddlewareOptionsSection": {
             "Param1": "Param1Value",
             "Param2": "Param2Value"
           }
         }

         ````

2. Retrieve options values. The `Get` method on the `Configuration` property lets you retrieve options values:

      <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Startup.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [12, 13, 14, 15, 16], "linenostart": 1}} -->

      ````c#

         // ASP.NET 5 Startup class

         namespace Asp.Net5
         {
             public class Startup
             {
                 public void Configure(IApplicationBuilder app, IHostingEnvironment env, 
                     ILoggerFactory loggerFactory)
                 {
                     // ...

                     var myMiddlewareOptions = 
                         Configuration.Get<MyMiddlewareOptions>("MyMiddlewareOptionsSection");

                     var myMiddlewareOptions2 = 
                         Configuration.Get<MyMiddlewareOptions>("MyMiddlewareOptionsSection2");

                     // ...

                 }
             }
         }

         ````

3. Pass options values to middleware. The `Use...` extension method (which adds your middleware to the pipeline) is a logical place to pass in the option values:

      <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Startup.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [18, 19, 20, 21, 22], "linenostart": 1}} -->

      ````c#

         // ASP.NET 5 Startup class

         namespace Asp.Net5
         {
             public class Startup
             {
                 public void Configure(IApplicationBuilder app, IHostingEnvironment env, 
                     ILoggerFactory loggerFactory)
                 {
                     // ...

                     var myMiddlewareOptions = 
                         Configuration.Get<MyMiddlewareOptions>("MyMiddlewareOptionsSection");

                     var myMiddlewareOptions2 = 
                         Configuration.Get<MyMiddlewareOptions>("MyMiddlewareOptionsSection2");

                     app.UseMyMiddlewareWithParams(myMiddlewareOptions);

                     // ...

                     app.UseMyMiddlewareWithParams(myMiddlewareOptions2);
                 }
             }
         }

         ````

4. Enable middleware to take an options parameter. Provide an overload of the `Use...` extension method (that takes the options parameter and passes it to `UseMiddleware`). When `UseMiddleware` is called with parameters, it passes the parameters to your middleware constructor when it instantiates the middleware object.

      <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/MyMiddlewareWithParams.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"hl_lines": [17, 18, 19, 20, 21, 22], "linenostart": 1}} -->

      ````c#

         using Microsoft.AspNet.Builder;
         using Microsoft.AspNet.Http;
         using Microsoft.Extensions.OptionsModel;
         using System.Threading.Tasks;

         namespace MyApp.Middleware
         {

             public static class MyMiddlewareWithParamsExtensions
             {
                 public static IApplicationBuilder UseMyMiddlewareWithParams(
                     this IApplicationBuilder builder)
                 {
                     return builder.UseMiddleware<MyMiddlewareWithParams>();
                 }

                 public static IApplicationBuilder UseMyMiddlewareWithParams(
                     this IApplicationBuilder builder, MyMiddlewareOptions myMiddlewareOptions)
                 {
                     return builder.UseMiddleware<MyMiddlewareWithParams>(
                         new OptionsWrapper<MyMiddlewareOptions>(myMiddlewareOptions));
                 }
             }
         }

         ````

      Note how this wraps the options object in an `OptionsWrapper` object. This implements `IOptions`, as expected by the middleware constructor:

      <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/MyMiddlewareWithParams.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "language": "c#", "highlight_args": {"linenostart": 1}} -->

      ````c#

             // Remove this when Microsoft.Extensions.Options becomes available from NuGet
             public class OptionsWrapper<TOptions> : IOptions<TOptions> where TOptions : class, new()
             {
                 public OptionsWrapper(TOptions options)
                 {
                     Value = options;
                 }

                 public TOptions Value { get; }
             }

         ````

## Migrating to the new HttpContext

You saw earlier that the `Invoke` method in your middleware takes a parameter of type `HttpContext`:

<!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

````c#

   public async Task Invoke(HttpContext context)
   ````

`HttpContext` has significantly changed in ASP.NET Core. This section shows how to translate the most commonly used properties of [System.Web.HttpContext](https://msdn.microsoft.com/en-us/library/system.web.httpcontext(v=vs.110).aspx) to the new [Microsoft.AspNetCore.Http.HttpContext](https://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/HttpContext/index.html).

### HttpContext

**HttpContext.Items** translates to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      IDictionary<object, object> items = httpContext.Items;

      ````

**Unique request ID (no System.Web.HttpContext counterpart)**

Gives you a unique id for each request. Very useful to include in your logs.

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      string requestId = httpContext.TraceIdentifier;

      ````

### HttpContext.Request

**HttpContext.Request.HttpMethod** translates to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      string httpMethod = httpContext.Request.Method;

      ````

**HttpContext.Request.QueryString** translates to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      IReadableStringCollection queryParameters = httpContext.Request.Query;

      // If no query parameter "key" used, values will have 0 items
      // If single value used for a key (...?key=v1), values will have 1 item ("v1")
      // If key has multiple values (...?key=v1&key=v2), values will have 2 items ("v1" and "v2")
      IList<string> values = queryParameters["key"];

      // If no query parameter "key" used, value will be ""
      // If single value used for a key (...?key=v1), value will be "v1"
      // If key has multiple values (...?key=v1&key=v2), value will be "v1,v2"
      string value = queryParameters["key"].ToString();

      ````

**HttpContext.Request.Url and HttpContext.Request.RawUrl** translate to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      // using Microsoft.AspNet.Http.Extensions;
      var url = httpContext.Request.GetDisplayUrl();

      ````

**HttpContext.Request.IsSecureConnection** translates to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      var isSecureConnection = httpContext.Request.IsHttps;

      ````

**HttpContext.Request.UserHostAddress** translates to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      var userHostAddress = httpContext.Connection.RemoteIpAddress?.ToString();

      ````

**HttpContext.Request.Cookies** translates to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      IReadableStringCollection cookies = httpContext.Request.Cookies;
      string unknownCookieValue = cookies["unknownCookie"]; // will be null (no exception)
      string knownCookieValue = cookies["cookie1name"];     // will be actual value

      ````

**HttpContext.Request.Headers** translates to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      // using Microsoft.AspNet.Http.Headers;
      // using Microsoft.Net.Http.Headers;

      IHeaderDictionary headersDictionary = httpContext.Request.Headers;

      // GetTypedHeaders extension method provides strongly typed access to many headers
      var requestHeaders = httpContext.Request.GetTypedHeaders();
      CacheControlHeaderValue cacheControlHeaderValue = requestHeaders.CacheControl;

      // For unknown header, unknownheaderValues has zero items and unknownheaderValue is ""
      IList<string> unknownheaderValues = headersDictionary["unknownheader"];
      string unknownheaderValue = headersDictionary["unknownheader"].ToString();

      // For known header, knownheaderValues has 1 item and knownheaderValue is the value
      IList<string> knownheaderValues = headersDictionary[HeaderNames.AcceptLanguage];
      string knownheaderValue = headersDictionary[HeaderNames.AcceptLanguage].ToString();

      ````

**HttpContext.Request.UserAgent** translates to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      string userAgent = headersDictionary[HeaderNames.UserAgent].ToString();

      ````

**HttpContext.Request.UrlReferrer** translates to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      string urlReferrer = headersDictionary[HeaderNames.Referer].ToString();

      ````

**HttpContext.Request.ContentType** translates to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      // using Microsoft.Net.Http.Headers;

      MediaTypeHeaderValue mediaHeaderValue = requestHeaders.ContentType;
      string contentType = mediaHeaderValue?.MediaType;   // ex. application/x-www-form-urlencoded
      string contentMainType = mediaHeaderValue?.Type;    // ex. application
      string contentSubType = mediaHeaderValue?.SubType;  // ex. x-www-form-urlencoded

      System.Text.Encoding requestEncoding = mediaHeaderValue?.Encoding;

      ````

**HttpContext.Request.Form** translates to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      if (httpContext.Request.HasFormContentType)
      {
          IFormCollection form;

          form = httpContext.Request.Form; // sync
          // Or
          form = await httpContext.Request.ReadFormAsync(); // async

          string firstName = form["firstname"];
          string lastName = form["lastname"];
      }

      ````

Caution: Read form values only if the content sub type is *x-www-form-urlencoded* or *form-data*.

**HttpContext.Request.InputStream** translates to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      string inputBody;
      using (var reader = new System.IO.StreamReader(
          httpContext.Request.Body, System.Text.Encoding.UTF8))
      {
          inputBody = reader.ReadToEnd();
      }

      ````

Caution: Use this code only in a handler type middleware, at the end of a pipeline.You can read the raw body as shown above only once per request. Middleware trying to read the body after the first read will read an empty body.This does not apply to reading a form as shown earlier, because that is done from a buffer.

**HttpContext.Request.RequestContext.RouteData**

RouteData is not available in middleware in RC1.

### HttpContext.Response

**HttpContext.Response.Status and HttpContext.Response.StatusDescription** translate to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      // using Microsoft.AspNet.Http;
      httpContext.Response.StatusCode = StatusCodes.Status200OK;

      ````

**HttpContext.Response.ContentEncoding and HttpContext.Response.ContentType** translate to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      // using Microsoft.Net.Http.Headers;
      var mediaType = new MediaTypeHeaderValue("application/json");
      mediaType.Encoding = System.Text.Encoding.UTF8;
      httpContext.Response.ContentType = mediaType.ToString();

      ````

**HttpContext.Response.ContentType** on its own also translates to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      httpContext.Response.ContentType = "text/html";

      ````

**HttpContext.Response.Output** translates to:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      string responseContent = GetResponseContent();
      await httpContext.Response.WriteAsync(responseContent);

      ````

**HttpContext.Response.TransmitFile**

Serving up a file is discussed [here](../fundamentals/request-features.html#middleware-and-request-features).

**HttpContext.Response.Headers**

Sending response headers is complicated by the fact that if you set them after anything has been written to the response body, they will not be sent.

The solution is to set a callback method that will be called right before writing to the response starts. This is best done at the start of the `Invoke` method in your middleware. It is this callback method that sets your response headers.

The following code sets a callback method called `SetHeaders`:

   <!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

   ````c#

      public async Task Invoke(HttpContext httpContext)
      {
          // ...
          httpContext.Response.OnStarting(SetHeaders, state: httpContext);
      ````

The `SetHeaders` callback method would look like this:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      // using Microsoft.AspNet.Http.Headers;
      // using Microsoft.Net.Http.Headers;

      private Task SetHeaders(object context)
      {
          var httpContext = (HttpContext)context;

          // Set header with single value
          httpContext.Response.Headers["ResponseHeaderName"] = "headerValue";

          // Set header with multiple values
          string[] responseHeaderValues = new string[] { "headerValue1", "headerValue1" };
          httpContext.Response.Headers["ResponseHeaderName"] = responseHeaderValues;

          // Translating ASP.NET 4's HttpContext.Response.RedirectLocation  
          httpContext.Response.Headers[HeaderNames.Location] = "http://www.example.com";
          // Or
          httpContext.Response.Redirect("http://www.example.com");

          // GetTypedHeaders extension method provides strongly typed access to many headers
          var responseHeaders = httpContext.Response.GetTypedHeaders();

          // Translating ASP.NET 4's HttpContext.Response.CacheControl 
          responseHeaders.CacheControl = new CacheControlHeaderValue
          {
              MaxAge = new System.TimeSpan(365, 0, 0, 0)
              // Many more properties available 
          };

          // If you use .Net 4.6+, Task.CompletedTask will be a bit faster
          return Task.FromResult(0);
      }

      ````

**HttpContext.Response.Cookies**

Cookies travel to the browser in a *Set-Cookie* response header. As a result, sending cookies requires the same callback as used for sending response headers:

   <!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

   ````c#

      public async Task Invoke(HttpContext httpContext)
      {
          // ...
          httpContext.Response.OnStarting(SetCookies, state: httpContext);
          httpContext.Response.OnStarting(SetHeaders, state: httpContext);
      ````

The `SetCookies` callback method would look like the following:

   <!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "migration/http-modules/sample/Asp.Net5/src/Asp.Net5/Middleware/HttpContextDemoMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      private Task SetCookies(object context)
      {
          var httpContext = (HttpContext)context;

          IResponseCookies responseCookies = httpContext.Response.Cookies;

          responseCookies.Append("cookie1name", "cookie1value");
          responseCookies.Append("cookie2name", "cookie2value",
              new CookieOptions { Expires = System.DateTime.Now.AddDays(5), HttpOnly = true });

          // If you use .Net 4.6+, Task.CompletedTask will be a bit faster
          return Task.FromResult(0); 
      }

      ````

## Additional Resources

* [HTTP Handlers and HTTP Modules Overview](https://msdn.microsoft.com/en-us/library/bb398986.aspx)

* [Configuration](../fundamentals/configuration.md)

* [Application Startup](../fundamentals/startup.md)

* [Middleware](../fundamentals/middleware.md)
