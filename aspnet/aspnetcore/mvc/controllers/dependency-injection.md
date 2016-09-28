---
uid: mvc/controllers/dependency-injection
---
<a name=dependency-injection-controllers></a>

# Dependency Injection and Controllers

By [Steve Smith](http://ardalis.com)

ASP.NET Core MVC controllers should request their dependencies explicitly via their constructors. In some instances, individual controller actions may require a service, and it may not make sense to request at the controller level. In this case, you can also choose to inject a service as a parameter on the action method.

[View or download sample code](https://github.com/aspnet/Docs/tree/master/aspnet/mvc/controllers/dependency-injection/sample)

## Dependency Injection

Dependency injection is a technique that follows the [Dependency Inversion Principle](http://deviq.com/dependency-inversion-principle), allowing for applications to be composed of loosely coupled modules. ASP.NET Core has built-in support for [dependency injection](../../fundamentals/dependency-injection.md), which makes applications easier to test and maintain.

## Constructor Injection

ASP.NET Core's built-in support for constructor-based dependency injection extends to MVC controllers. By simply adding a service type to your controller as a constructor parameter, ASP.NET Core will attempt to resolve that type using its built in service container. Services are typically, but not always, defined using interfaces. For example, if your application has business logic that depends on the current time, you can inject a service that retrieves the time (rather than hard-coding it), which would allow your tests to pass in implementations that use a set time.

<!-- literal_block {"xml:space": "preserve", "source": "mvc/controllers/dependency-injection/sample/src/ControllerDI/Interfaces/IDateTime.cs", "ids": [], "linenos": false, "language": "csharp", "highlight_args": {"linenostart": 1}} -->

````c#

   using System;

   namespace ControllerDI.Interfaces
   {
       public interface IDateTime
       {
           DateTime Now { get; }
       }
   }

   ````

Implementing an interface like this one so that it uses the system clock at runtime is trivial:

<!-- literal_block {"xml:space": "preserve", "source": "mvc/controllers/dependency-injection/sample/src/ControllerDI/Services/SystemDateTime.cs", "ids": [], "linenos": false, "language": "csharp", "highlight_args": {"linenostart": 1}} -->

````c#

   using System;
   using ControllerDI.Interfaces;

   namespace ControllerDI.Services
   {
       public class SystemDateTime : IDateTime
       {
           public DateTime Now
           {
               get { return DateTime.Now; }
           }
       }
   }

   ````

With this in place, we can use the service in our controller. In this case, we have added some logic to the `HomeController` `Index` method to display a greeting to the user based on the time of day.

[!code-csharp[Main](../mvc/controllers/dependency-injection/sample/src/ControllerDI/Controllers/HomeController.cs?highlight=8,10,12,17,18,19,20,21,22,23,24,25,26,27,28,29,30)]

````c#

   using ControllerDI.Interfaces;
   using Microsoft.AspNetCore.Mvc;

   namespace ControllerDI.Controllers
   {
       public class HomeController : Controller
       {
           private readonly IDateTime _dateTime;

           public HomeController(IDateTime dateTime)
           {
               _dateTime = dateTime;
           }

           public IActionResult Index()
           {
               var serverTime = _dateTime.Now;
               if (serverTime.Hour < 12)
               {
                   ViewData["Message"] = "It's morning here - Good Morning!";
               }
               else if (serverTime.Hour < 17)
               {
                   ViewData["Message"] = "It's afternoon here - Good Afternoon!";
               }
               else
               {
                   ViewData["Message"] = "It's evening here - Good Evening!";
               }
               return View();
           }
       }
   }

   ````

If we run the application now, we will most likely encounter an error:

<!-- literal_block {"ids": [], "xml:space": "preserve"} -->

````

   An unhandled exception occurred while processing the request.

   InvalidOperationException: Unable to resolve service for type 'ControllerDI.Interfaces.IDateTime' while attempting to activate 'ControllerDI.Controllers.HomeController'.
   Microsoft.Extensions.DependencyInjection.ActivatorUtilities.GetService(IServiceProvider sp, Type type, Type requiredBy, Boolean isDefaultParameterRequired)
   ````

This error occurs when we have not configured a service in the `ConfigureServices` method in our `Startup` class. To specify that requests for `IDateTime` should be resolved using an instance of `SystemDateTime`, add the highlighted line in the listing below to your `ConfigureServices` method:

[!code-csharp[Main](../mvc/controllers/dependency-injection/sample/src/ControllerDI/Startup.cs?highlight=4)]

````c#

   public void ConfigureServices(IServiceCollection services)
   {
       // Add application services.
       services.AddTransient<IDateTime, SystemDateTime>();
   }

   ````

> [!NOTE]
> This particular service could be implemented using any of several different lifetime options (`Transient`, `Scoped`, or `Singleton`). See [Dependency Injection](../../fundamentals/dependency-injection.md) to understand how each of these scope options will affect the behavior of your service.

Once the service has been configured, running the application and navigating to the home page should display the time-based message as expected:

![image](dependency-injection/_static/server-greeting.png)

>[!TIP]
> See [Testing Controller Logic](testing.md) to learn how to explicitly request dependencies [http://deviq.com/explicit-dependencies-principle](http://deviq.com/explicit-dependencies-principle) in controllers makes code easier to test.

ASP.NET Core's built-in dependency injection supports having only a single constructor for classes requesting services. If you have more than one constructor, you may get an exception stating:

<!-- literal_block {"ids": [], "xml:space": "preserve"} -->

````

   An unhandled exception occurred while processing the request.

   InvalidOperationException: Multiple constructors accepting all given argument types have been found in type 'ControllerDI.Controllers.HomeController'. There should only be one applicable constructor.
   Microsoft.Extensions.DependencyInjection.ActivatorUtilities.FindApplicableConstructor(Type instanceType, Type[] argumentTypes, ConstructorInfo& matchingConstructor, Nullable`1[]& parameterMap)
   ````

As the error message states, you can correct this problem having just a single constructor. You can also [replace the default dependency injection support with a third party implementation](../../fundamentals/dependency-injection.md#replacing-the-default-services-container.md), many of which support multiple constructors.

## Action Injection with FromServices

Sometimes you don't need a service for more than one action within your controller. In this case, it may make sense to inject the service as a parameter to the action method. This is done by marking the parameter with the attribute `[FromServices]` as shown here:

[!code-csharp[Main](../mvc/controllers/dependency-injection/sample/src/ControllerDI/Controllers/HomeController.cs?highlight=1)]

````c#

   public IActionResult About([FromServices] IDateTime dateTime)
   {
       ViewData["Message"] = "Currently on the server the time is " + dateTime.Now;

       return View();
   }

   ````

## Accessing Settings from a Controller

Accessing application or configuration settings from within a controller is a common pattern. This access should use the Options pattern described in [configuration](../../fundamentals/configuration.md). You generally should not request settings directly from your controller using dependency injection. A better approach is to request an `IOptions<T>` instance, where `T` is the configuration class you need.

To work with the options pattern, you need to create a class that represents the options, such as this one:

<!-- literal_block {"xml:space": "preserve", "source": "mvc/controllers/dependency-injection/sample/src/ControllerDI/Model/SampleWebSettings.cs", "ids": [], "linenos": false, "language": "csharp", "highlight_args": {"linenostart": 1}} -->

````c#

   namespace ControllerDI.Model
   {
       public class SampleWebSettings
       {
           public string Title { get; set; }
           public int Updates { get; set; }
       }
   }

   ````

Then you need to configure the application to use the options model and add your configuration class to the services collection in `ConfigureServices`:

[!code-csharp[Main](../mvc/controllers/dependency-injection/sample/src/ControllerDI/Startup.cs?highlight=3,4,5,6,9,16,19)]

````c#

   public Startup(IHostingEnvironment env)
   {
       var builder = new ConfigurationBuilder()
           .SetBasePath(env.ContentRootPath)
           .AddJsonFile("samplewebsettings.json");
       Configuration = builder.Build();
   }

   public IConfigurationRoot Configuration { get; set; }

   // This method gets called by the runtime. Use this method to add services to the container.
   // For more information on how to configure your application, visit http://go.microsoft.com/fwlink/?LinkID=398940
   public void ConfigureServices(IServiceCollection services)
   {
       // Required to use the Options<T> pattern
       services.AddOptions();

       // Add settings from configuration
       services.Configure<SampleWebSettings>(Configuration);

       // Uncomment to add settings from code
       //services.Configure<SampleWebSettings>(settings =>
       //{
       //    settings.Updates = 17;
       //});

       services.AddMvc();

       // Add application services.
       services.AddTransient<IDateTime, SystemDateTime>();
   }

   ````

> [!NOTE]
> In the above listing, we are configuring the application to read the settings from a JSON-formatted file. You can also configure the settings entirely in code, as is shown in the commented code above. See [Configuration](../../fundamentals/configuration.md) for further configuration options.

Once you've specified a strongly-typed configuration object (in this case, `SampleWebSettings`) and added it to the services collection, you can request it from any Controller or Action method by requesting an instance of `IOptions<T>` (in this case, `IOptions<SampleWebSettings>`). The following code shows how one would request the settings from a controller:

[!code-csharp[Main](../mvc/controllers/dependency-injection/sample/src/ControllerDI/Controllers/SettingsController.cs?highlight=3,5,7)]

````c#

       public class SettingsController : Controller
       {
           private readonly SampleWebSettings _settings;

           public SettingsController(IOptions<SampleWebSettings> settingsOptions)
           {
               _settings = settingsOptions.Value;
           }

           public IActionResult Index()
           {
               ViewData["Title"] = _settings.Title;
               ViewData["Updates"] = _settings.Updates;
               return View();
           }
       }

   ````

Following the Options pattern allows settings and configuration to be decoupled from one another, and ensures the controller is following [separation of concerns](http://deviq.com/separation-of-concerns/), since it doesn't need to know how or where to find the settings information. It also makes the controller easier to unit test [Testing Controller Logic](testing.md), since there is no [static cling](http://deviq.com/static-cling/) or direct instantiation of settings classes within the controller class.
