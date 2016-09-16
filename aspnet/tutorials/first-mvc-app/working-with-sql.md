---
uid: tutorials/first-mvc-app/working-with-sql
---
  # Working with SQL Server LocalDB

By [Rick Anderson](https://twitter.com/RickAndMSFT)

The `ApplicationDbContext` class handles the task of connecting to the database and mapping `Movie` objects to database records. The database context is registered with the [Dependency Injection](../../fundamentals/dependency-injection.md) container in the `ConfigureServices` method in the *Startup.cs* file:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/tutorials/first-mvc-app/start-mvc/sample2/src/MvcMovie/Startup.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

````c#

   public void ConfigureServices(IServiceCollection services)
   {
       // Add framework services.
       services.AddDbContext<ApplicationDbContext>(options =>
           options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));

   ````

The ASP.NET Core [Configuration](../../fundamentals/configuration.md) system reads the `ConnectionString`. For local development, it gets the connection string from the *appsettings.json* file:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/tutorials/first-mvc-app/start-mvc/sample2/src/MvcMovie/appsettings.json", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "javascript", "highlight_args": {"hl_lines": [3], "linenostart": 1}} -->

````javascript

   {
     "ConnectionStrings": {
       "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=aspnet-MvcMovie-4ae3798a;Trusted_Connection=True;MultipleActiveResultSets=true"
     },
     "Logging": {
       "IncludeScopes": false,

   ````

When you deploy the app to a test or production server, you can use an environment variable or another approach to set the connection string to a real SQL Server. See [Configuration](../../fundamentals/configuration.md) .

  ## SQL Server Express LocalDB

LocalDB is a lightweight version of the SQL Server Express Database Engine that is targeted for program development. LocalDB starts on demand and runs in user mode, so there is no complex configuration. By default, LocalDB database creates "*.mdf" files in the *C:/Users/<user>* directory.

* From the **View** menu, open **SQL Server Object Explorer** (SSOX).

![image](working-with-sql/_static/ssox.png)

* Right click on the `Movie` table **> View Designer**

![image](working-with-sql/_static/design.png)

![image](working-with-sql/_static/dv.png)

Note the key icon next to `ID`. By default, EF will make a property named `ID` the primary key.

* Right click on the `Movie` table **> View Data**

![image](working-with-sql/_static/ssox2.png)

![image](working-with-sql/_static/vd22.png)

  ## Seed the database

Create a new class named `SeedData` in the *Models* folder. Replace the generated code with the following:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/tutorials/first-mvc-app/start-mvc/sample2/src/MvcMovie/Models/SeedData.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

````c#

   using Microsoft.EntityFrameworkCore;
   using Microsoft.Extensions.DependencyInjection;
   using MvcMovie.Data;
   using System;
   using System.Linq;

   namespace MvcMovie.Models
   {
       public static class SeedData
       {
           public static void Initialize(IServiceProvider serviceProvider)
           {
               using (var context = new ApplicationDbContext(
                   serviceProvider.GetRequiredService<DbContextOptions<ApplicationDbContext>>()))
               {
                   // Look for any movies.
                   if (context.Movie.Any())
                   {
                       return;   // DB has been seeded
                   }

                   context.Movie.AddRange(
                        new Movie
                        {
                            Title = "When Harry Met Sally",
                            ReleaseDate = DateTime.Parse("1989-1-11"),
                            Genre = "Romantic Comedy",
                            Price = 7.99M
                        },

                        new Movie
                        {
                            Title = "Ghostbusters ",
                            ReleaseDate = DateTime.Parse("1984-3-13"),
                            Genre = "Comedy",
                            Price = 8.99M
                        },

                        new Movie
                        {
                            Title = "Ghostbusters 2",
                            ReleaseDate = DateTime.Parse("1986-2-23"),
                            Genre = "Comedy",
                            Price = 9.99M
                        },

                      new Movie
                      {
                          Title = "Rio Bravo",
                          ReleaseDate = DateTime.Parse("1959-4-15"),
                          Genre = "Western",
                          Price = 3.99M
                      }
                   );
                   context.SaveChanges();
               }
           }
       }
   }

   ````

Notice if there are any movies in the DB, the seed initializer returns.

<!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

````c#

   if (context.Movie.Any())
   {
       return;   // DB has been seeded.
   }
   ````

Add the seed initializer to the end of the `Configure` method in the *Startup.cs* file:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/tutorials/first-mvc-app/start-mvc/sample2/src/MvcMovie/Startup.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "highlight_args": {"hl_lines": [9], "linenostart": 1}} -->

````

       app.UseMvc(routes =>
       {
           routes.MapRoute(
               name: "default",
               template: "{controller=Home}/{action=Index}/{id?}");
       });
       #endregion

       SeedData.Initialize(app.ApplicationServices);
   }
   // End of Configure.



   ````

Test the app

* Delete all the records in the DB. You can do this with the delete links in the browser or from SSOX.

* Force the app to initialize (call the methods in the `Startup` class) so the seed method runs. To force initialization, IIS Express must be stopped and restarted. You can do this with any of the following approaches:

  * Right click the IIS Express system tray icon in the notification area and tap **Exit** or **Stop Site**



![image](working-with-sql/_static/iisExIcon.png)



![image](working-with-sql/_static/stopIIS.png)



   * If you were running VS in non-debug mode, press F5 to run in debug mode

   * If you were running VS in debug mode, stop the debugger and press ^F5

Note: If the database doesn't initialize, put a break point on the line `if (context.Movie.Any())` and start debugging.

![image](working-with-sql/_static/dbg.png)

The app shows the seeded data.

![image](working-with-sql/_static/m55.png)
