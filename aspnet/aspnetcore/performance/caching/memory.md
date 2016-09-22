---
uid: performance/caching/memory
---
Warning: This page documents version 1.0.0-rc2 and has not yet been updated for version 1.0.0

# In Memory Caching

By [Steve Smith](http://ardalis.com)

Caching involves keeping a copy of data in a location that can be accessed more quickly than the source data. ASP.NET Core has rich support for caching in a variety of ways, including keeping data in memory on the local server, which is referred to as *in memory caching*.

[View or download sample code](https://github.com/aspnet/Docs/tree/master/aspnet/performance/caching/memory/sample)

<a name=caching-basics></a>

## Caching Basics

Caching can dramatically improve the performance and scalability of ASP.NET applications, by eliminating unnecessary requests to external data sources for data that changes infrequently.

> [!NOTE]
> Caching in all forms (in-memory or distributed, including session state) involves making a copy of data in order to optimize performance. The copied data should be considered ephemeral - it could disappear at any time. Apps should be written to not depend on cached data, but use it when available.

ASP.NET supports several different kinds of caches, the simplest of which is represented by the [IMemoryCache](https://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/Extensions/Caching/Memory/IMemoryCache/index.html) interface, which represents a cache stored in the memory of the local web server.

You should always write (and test!) your application such that it can use cached data if it's available, but otherwise will work correctly using the underlying data source.

An in-memory cache is stored in the memory of a single server hosting an ASP.NET app. If an app is hosted by multiple servers in a web farm or cloud hosting environment, the servers may have different values in their local in-memory caches. Apps that will be hosted in server farms or on cloud hosting should use a [distributed cache](distributed.md) to avoid cache consistency problems.

Tip: A common use case for caching is data-driven navigation menus, which rarely change but are frequently read for display within an application. Caching results that do not vary often but which are requested frequently can greatly improve performance by reducing round trips to out of process data stores and unnecessary computation.

## Configuring In Memory Caching

To use an in memory cache in your ASP.NET application, add the following dependencies to your *project.json* file:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/performance/caching/memory/sample/src/CachingSample/project.json", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "highlight_args": {"hl_lines": [4], "linenostart": 1}} -->

````

     "dependencies": {
       "Microsoft.AspNetCore.Server.IISIntegration": "1.0.0-rc2-final",
       "Microsoft.AspNetCore.Server.Kestrel": "1.0.0-rc2-final",
       "Microsoft.Extensions.Caching.Memory": "1.0.0-rc2-final",
       "Microsoft.Extensions.Logging": "1.0.0-rc2-final",
       "Microsoft.Extensions.Logging.Console": "1.0.0-rc2-final"
     },

   ````

Caching in ASP.NET Core is a *service* that should be referenced from your application by [Dependency Injection](../../fundamentals/dependency-injection.md). To register the caching service and make it available within your app, add the following line to your `ConfigureServices` method in `Startup`:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/performance/caching/memory/sample/src/CachingSample/Startup.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "highlight_args": {"hl_lines": [3], "linenostart": 1}} -->

````

   public void ConfigureServices(IServiceCollection services)
   {
       services.AddMemoryCache();


   ````

You utilize caching in your app by requesting an instance of `IMemoryCache` in your controller or middleware constructor. In the sample for this article, we are using a simple middleware component to handle requests by returning customized greeting. The constructor is shown here:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/performance/caching/memory/sample/src/CachingSample/Middleware/GreetingMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "highlight_args": {"hl_lines": [2, 7], "linenostart": 1}} -->

````

   public GreetingMiddleware(RequestDelegate next,
       IMemoryCache memoryCache,
       ILogger<GreetingMiddleware> logger,
       IGreetingService greetingService)
   {
       _next = next;
       _memoryCache = memoryCache;
       _greetingService = greetingService;
       _logger = logger;
   }

   ````

## Reading and Writing to a Memory Cache

The middleware's `Invoke` method returns the cached data when it's available.

There are two methods for accessing cache entries:

`Get`
   `Get` will return the value if it exists, but otherwise returns `null`.

`TryGet`
   `TryGet` will assign the cached value to an `out` parameter and return true if the entry exists. Otherwise it returns false.

Use the `Set` method to write to the cache. `Set` accepts the key to use to look up the value, the value to be cached, and a set of `MemoryCacheEntryOptions`. The `MemoryCacheEntryOptions` allow you to specify absolute or sliding time-based cache expiration, caching priority, callbacks, and dependencies. These options are detailed below.

The sample code (shown below) uses the `SetAbsoluteExpiration` method on `MemoryCacheEntryOptions` to cache greetings for one minute.

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/performance/caching/memory/sample/src/CachingSample/Middleware/GreetingMiddleware.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "highlight_args": {"hl_lines": [7, 10, 16, 17, 18], "linenostart": 1}} -->

````

   public Task Invoke(HttpContext httpContext)
   {
       string cacheKey = "GreetingMiddleware-Invoke";
       string greeting;

       // try to get the cached item; null if not found
       // greeting = _memoryCache.Get(cacheKey) as string;

       // alternately, TryGet returns true if the cache entry was found
       if(!_memoryCache.TryGetValue(cacheKey, out greeting))
       {
           // fetch the value from the source
           greeting = _greetingService.Greet("world");

           // store in the cache
           _memoryCache.Set(cacheKey, greeting,
               new MemoryCacheEntryOptions()
               .SetAbsoluteExpiration(TimeSpan.FromMinutes(1)));
           _logger.LogInformation($"{cacheKey} updated from source.");
       }
       else
       {
           _logger.LogInformation($"{cacheKey} retrieved from cache.");
       }

       return httpContext.Response.WriteAsync(greeting);
   }



   ````

In addition to setting an absolute expiration, a sliding expiration can be used to keep frequently requested items in the cache:

<!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

````c#

   // keep item in cache as long as it is requested at least
   // once every 5 minutes
   new MemoryCacheEntryOptions()
     .SetSlidingExpiration(TimeSpan.FromMinutes(5))
   ````

To avoid having frequently-accessed cache entries growing too stale (because their sliding expiration is constantly reset), you can combine absolute and sliding expirations:

<!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

````c#

   // keep item in cache as long as it is requested at least
   // once every 5 minutes...
   // but in any case make sure to refresh it every hour
   new MemoryCacheEntryOptions()
     .SetSlidingExpiration(TimeSpan.FromMinutes(5))
     .SetAbsoluteExpiration(TimeSpan.FromHours(1))
   ````

By default, an instance of `MemoryCache` will automatically manage the items stored, removing entries when necessary in response to memory pressure in the app. You can influence the way cache entries are managed by setting their [CacheItemPriority](https://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/Extensions/Caching/Memory/CacheItemPriority/index.html) when adding the item to the cache. For instance, if you have an item you want to keep in the cache unless you explicitly remove it, you would use the `NeverRemove` priority option:

<!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

````c#

   // keep item in cache indefinitely unless explicitly removed
   new MemoryCacheEntryOptions()
     .SetPriority(CacheItemPriority.NeverRemove))
   ````

When you do want to explicitly remove an item from the cache, you can do so easily using the `Remove` method:

<!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

````c#

   cache.Remove(cacheKey);
   ````

## Cache Dependencies and Callbacks

You can configure cache entries to depend on other cache entries, the file system, or programmatic tokens, evicting the entry in response to changes. You can register a callback, which will run when a cache item is evicted.

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/performance/caching/memory/sample/test/CachingSample.Tests/MemoryCacheTests.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "highlight_args": {"hl_lines": [6, 7, 8, 9, 10, 11, 18], "linenostart": 1}} -->

````

   {
       var pause = new ManualResetEvent(false);

       _memoryCache.Set(_cacheKey, _cacheItem,
           new MemoryCacheEntryOptions()
           .RegisterPostEvictionCallback(
               (key, value, reason, substate) =>
               {
                   _result = $"'{key}':'{value}' was evicted because: {reason}";
                   pause.Set();
               }
           ));

       _memoryCache.Remove(_cacheKey);

       Assert.True(pause.WaitOne(500));

       Assert.Equal("'key':'value' was evicted because: Removed", _result);
   }


   ````

The callback is run on a different thread from the code that removes the item from the cache.

Warning: If the callback is used to repopulate the cache it is possible other requests for the cache will take place (and find it empty) before the callback completes, possibly resulting in several threads repopulating the cached value.

Possible [eviction reasons](https://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/Extensions/Caching/Memory/EvictionReason/index.html) are:

None
   No reason known.

Removed
   The item was manually removed by a call to `Remove()`

Replaced
   The item was overwritten.

Expired
   The item timed out.

TokenExpired
   The token the item depended upon fired an event.

Capacity
   The item was removed as part of the cache's memory management process.

You can specify that one or more cache entries depend on a `CancellationTokenSource` by adding the expiration token to the `MemoryCacheEntryOptions` object. When a cached item is invalidated, call `Cancel` on the token, which will expire all of the associated cache entries (with a reason of `TokenExpired`). The following unit test demonstrates this:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/performance/caching/memory/sample/test/CachingSample.Tests/MemoryCacheTests.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "highlight_args": {"hl_lines": [7, 16, 21], "linenostart": 1}} -->

````

   public void CancellationTokenFiresCallback()
   {
       var cts = new CancellationTokenSource();
       var pause = new ManualResetEvent(false);
       _memoryCache.Set(_cacheKey, _cacheItem,
           new MemoryCacheEntryOptions()
           .AddExpirationToken(new CancellationChangeToken(cts.Token))
           .RegisterPostEvictionCallback(
               (key, value, reason, substate) =>
               {
                   _result = $"'{key}':'{value}' was evicted because: {reason}";
                   pause.Set();
               }
           ));

       // trigger the token
       cts.Cancel();

       Assert.True(pause.WaitOne(500));

       Assert.Equal("'key':'value' was evicted because: TokenExpired", _result);
   }

   ````

Using a `CancellationTokenSource` allows multiple cache entries to all be expired without the need to create a dependency between cache entries themselves (in which case, you must ensure that the source cache entry exists before it is used as a dependency for other entries).

Cache entries will inherit triggers and timeouts from other entries accessed while creating the new entry. This approach ensures that subordinate cache entries expire at the same time as related entries.

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/performance/caching/memory/sample/test/CachingSample.Tests/MemoryCacheTests.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": true, "highlight_args": {"hl_lines": [7, 11, 13, 23, 24], "linenostart": 1}} -->

````

   [Fact]
   public void CacheEntryDependencies()
   {
       var cts = new CancellationTokenSource();
       var pause = new ManualResetEvent(false);

       using (var cacheEntry = _memoryCache.CreateEntry(_cacheKey))
       {
           _memoryCache.Set("master key", "some value",
               new MemoryCacheEntryOptions()
               .AddExpirationToken(new CancellationChangeToken(cts.Token)));

           cacheEntry.SetValue(_cacheItem)
               .RegisterPostEvictionCallback(
                   (key, value, reason, substate) =>
                   {
                       _result = $"'{key}':'{value}' was evicted because: {reason}";
                       pause.Set();
                   }
               );
       }

       // trigger the token to expire the master item
       cts.Cancel();

       Assert.True(pause.WaitOne(500));

       Assert.Equal("'key':'value' was evicted because: TokenExpired", _result);
   }

   ````

> [!NOTE]
> When one cache entry is used to create another, the new one copies the existing entry's expiration tokens and time-based expiration settings, if any. It is not expired in response to manual removal or updating of the existing entry.

## Other Resources

* [Working with a Distributed Cache](distributed.md)
