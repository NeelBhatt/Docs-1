---
uid: getting-started
---
# Getting Started

1. Install [.NET Core](https://microsoft.com/net/core)

2. Create a new .NET Core project:


   ````console

      mkdir aspnetcoreapp
      cd aspnetcoreapp
      dotnet new
      ````

3. Update the *project.json* file to add the Kestrel HTTP server package as a dependency:

   [!code-csharp[Main](../getting-started/sample/aspnetcoreapp/project.json?highlight=15)]

   ````csharp

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


   ````console

      dotnet restore
      ````

5. Add a *Startup.cs* file that defines the request handling logic:

   <!-- literal_block {"xml:space": "preserve", "source": "getting-started/sample/aspnetcoreapp/Startup.cs", "ids": [], "linenos": false, "language": "csharp", "highlight_args": {"linenostart": 1}} -->

   ````csharp

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

   [!code-csharp[Main](../getting-started/sample/aspnetcoreapp/Program.cs?highlight=2,4,10,11,12,13,14,15)]

   ````csharp

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
