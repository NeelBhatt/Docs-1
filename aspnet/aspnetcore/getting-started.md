---
uid: getting-started
---
# Getting Started

1. Install [.NET Core](https://microsoft.com/net/core)

2. Create a new .NET Core project:

   <!-- literal_block {"ids": [], "linenos": false, "xml:space": "preserve", "language": "console"} -->

   ````console

      mkdir aspnetcoreapp
      cd aspnetcoreapp
      dotnet new
      ````

3. Update the *project.json* file to add the Kestrel HTTP server package as a dependency:

   <!-- literal_block {"xml:space": "preserve", "source": "getting-started/sample/aspnetcoreapp/project.json", "ids": [], "linenos": false, "language": "c#", "highlight_args": {"hl_lines": [15], "linenostart": 1}} -->

   ````c#

      {
        "version": "1.0.0-*",
        "buildOptions": {
          "debugType": "portable",
          "emitEntryPoint": true
        },
        "dependencies": {},
        "frameworks": {
          "netcoreapp1.0": {
            "dependencies": {
              "Microsoft.NETCore.App": {
                "type": "platform",
                "version": "1.0.0"
              },
              "Microsoft.AspNetCore.Server.Kestrel": "1.0.0"
            },
            "imports": "dnxcore50"
          }
        }
      }

      ````

4. Restore the packages:

   <!-- literal_block {"ids": [], "linenos": false, "xml:space": "preserve", "language": "console"} -->

   ````console

      dotnet restore
      ````

5. Add a *Startup.cs* file that defines the request handling logic:

   <!-- literal_block {"xml:space": "preserve", "source": "getting-started/sample/aspnetcoreapp/Startup.cs", "ids": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

   ````c#

      using System;
      using Microsoft.AspNetCore.Builder;
      using Microsoft.AspNetCore.Hosting;
      using Microsoft.AspNetCore.Http;

      namespace aspnetcoreapp
      {
          public class Startup
          {
              public void Configure(IApplicationBuilder app)
              {
                  app.Run(context =>
                  {
                      return context.Response.WriteAsync("Hello from ASP.NET Core!");
                  });
              }
          }
      }

      ````

6. Update the code in *Program.cs* to setup and start the Web host:

   <!-- literal_block {"xml:space": "preserve", "source": "getting-started/sample/aspnetcoreapp/Program.cs", "ids": [], "linenos": false, "language": "c#", "highlight_args": {"hl_lines": [2, 4, 10, 11, 12, 13, 14, 15], "linenostart": 1}} -->

   ````c#

      using System;
      using Microsoft.AspNetCore.Hosting;

      namespace aspnetcoreapp
      {
          public class Program
          {
              public static void Main(string[] args)
              {
                  var host = new WebHostBuilder()
                      .UseKestrel()
                      .UseStartup<Startup>()
                      .Build();

                  host.Run();
              }
          }
      }

      ````

7. Run the app  (the `dotnet run` command will build the app when it's out of date):

   <!-- literal_block {"ids": [], "linenos": false, "xml:space": "preserve", "language": "console"} -->

   ````console

      dotnet run
      ````

8. Browse to http://localhost:5000:

   ![image](getting-started/_static/running-output.png)

## Next steps

* [Building your first ASP.NET Core MVC app with Visual Studio](tutorials/first-mvc-app/index.md)

* [Your First ASP.NET Core Application on a Mac Using Visual Studio Code](tutorials/your-first-mac-aspnet.md)

* [Building Your First Web API with ASP.NET Core MVC and Visual Studio](tutorials/first-web-api.md)

* [Fundamentals](fundamentals/index.md)
