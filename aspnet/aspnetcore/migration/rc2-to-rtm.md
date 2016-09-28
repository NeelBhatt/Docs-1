---
uid: migration/rc2-to-rtm
---
# Migrating from ASP.NET Core RC2 to ASP.NET Core 1.0

By [Cesar Blum Silveira](https://github.com/cesarbs)

## Overview

This migration guide covers migrating an ASP.NET Core RC2 application to ASP.NET Core 1.0.

There weren't many significant changes to ASP.NET Core between the RC2 and 1.0 releases. For a complete list of changes, see the [ASP.NET Core 1.0 announcements](https://github.com/aspnet/announcements/issues?q=is%3Aopen+is%3Aissue+milestone%3A1.0.0).

Install the new tools from [https://dot.net/core](https://dot.net/core) and follow the instructions.

Update the global.json to

<!-- literal_block {"ids": [], "linenos": false, "xml:space": "preserve", "language": "javascript"} -->

````javascript

   {
     "projects": [ "src", "test" ],
     "sdk": {
         "version": "1.0.0-preview2-003121"
     }
   }
   ````

## Tools

For the tools we ship, you no longer need to use `imports` in *project.json*. For example:

<!-- literal_block {"ids": [], "linenos": false, "xml:space": "preserve", "language": "json"} -->

````json

   {
     "tools": {
       "Microsoft.AspNetCore.Server.IISIntegration.Tools": {
         "version": "1.0.0-preview1-final",
         "imports": "portable-net45+win8+dnxcore50"
       }
     }
   }
   ````

Becomes:

<!-- literal_block {"ids": [], "linenos": false, "xml:space": "preserve", "language": "json"} -->

````json

   {
     "tools": {
       "Microsoft.AspNetCore.Server.IISIntegration.Tools": "1.0.0-preview2-final"
     }
   }
   ````

## Hosting

The `UseServer` is no longer available for [IWebHostBuilder](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Hosting/IWebHostBuilder/index.html.md#Microsoft.AspNetCore.Hosting.IWebHostBuilder.md). You must now use [UseKestrel](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Hosting/WebHostBuilderKestrelExtensions/index.html.md#Microsoft.AspNetCore.Hosting.WebHostBuilderKestrelExtensions.UseKestrel.md) or [UseWebListener](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Hosting/WebHostBuilderWebListenerExtensions/index.html.md#Microsoft.AspNetCore.Hosting.WebHostBuilderWebListenerExtensions.UseWebListener.md).

## ASP.NET MVC Core

The `HtmlEncodedString` class has been replaced by [HtmlString](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Html/HtmlString/index.html.md#Microsoft.AspNetCore.Html.HtmlString.md) (contained in the  `Microsoft.AspNetCore.Html.Abstractions` package).

## Security

The [AuthorizationHandler<TRequirement>](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Authorization/AuthorizationHandler-TRequirement/index.html.md#Microsoft.AspNetCore.Authorization.AuthorizationHandler<TRequirement>.md) class now only contains an asynchronous interface.
