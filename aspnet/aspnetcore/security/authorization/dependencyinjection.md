---
uid: security/authorization/dependencyinjection
---
<a name=security-authorization-di></a>

# Dependency Injection in requirement handlers

[Authorization handlers must be registered](policies.md#security-authorization-policies-based-handler-registration.md) in the service collection during configuration (using [dependency injection](../../fundamentals/dependency-injection.md#fundamentals-dependency-injection.md)).

Suppose you had a repository of rules you wanted to evaluate inside an authorization handler and that repository was registered in the service collection.  Authorization will resolve and inject that into your constructor.

For example, if you wanted to use ASP.NET's logging infrastructure you would to inject [ILoggerFactory](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/Extensions/Logging/ILoggerFactory/index.html.md#Microsoft.Extensions.Logging.ILoggerFactory.md) into your handler. Such a handler might look like:

<!-- literal_block {"ids": [], "linenos": false, "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

````c#

   public class LoggingAuthorizationHandler : AuthorizationHandler<MyRequirement>
   {
       ILogger _logger;

       public LoggingAuthorizationHandler(ILoggerFactory loggerFactory)
       {
           _logger = loggerFactory.CreateLogger(this.GetType().FullName);
       }

       protected override Task HandleRequirementAsync(AuthorizationHandlerContext context, MyRequirement requirement)
       {
           _logger.LogInformation("Inside my handler");
           // Check if the requirement is fulfilled.
           return Task.CompletedTask;
       }
   }
   ````

You would register the handler with `services.AddSingleton()`:

<!-- literal_block {"ids": [], "linenos": false, "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

````c#

   services.AddSingleton<IAuthorizationHandler, LoggingAuthorizationHandler>();
   ````

An instance of the handler will be created when your application starts, and DI will inject the registered [ILoggerFactory](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/Extensions/Logging/ILoggerFactory/index.html.md#Microsoft.Extensions.Logging.ILoggerFactory.md) into your constructor.

> [!NOTE]
> Handlers that use Entity Framework shouldn't be registered as singletons.
