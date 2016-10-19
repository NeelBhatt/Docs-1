---
uid: data/ef-mvc/migrations
---
# Migrations

The Contoso University sample web application demonstrates how to create ASP.NET Core 1.0 MVC web applications using Entity Framework Core 1.0 and Visual Studio 2015. For information about the tutorial series, see [the first tutorial in the series](intro.md).

In this tutorial, you start using the EF Core migrations feature for managing data model changes. In later tutorials, you'll add more migrations as you change the data model.

## Introduction to migrations

When you develop a new application, your data model changes frequently, and each time the model changes, it gets out of sync with the database. You started these tutorials by configuring the Entity Framework to create the database if it doesn't exist. Then each time you change the data model -- add, remove, or change entity classes or change your DbContext class -- you can delete the database and EF creates a new one that matches the model, and seeds it with test data.

This method of keeping the database in sync with the data model works well until you deploy the application to production. When the application is running in production it is usually storing data that you want to keep, and you don't want to lose everything each time you make a change such as adding a new column. The EF Core Migrations feature solves this problem by enabling EF to update the database schema instead of creating  a new database.

## Change the connection string

In the *appsettings.json* file, change the name of the database in the connection string to ContosoUniversity2 or some other name that you haven't used on the computer you're using.

<!-- literal_block {"ids": [], "names": [], "highlight_args": {"hl_lines": [6], "linenostart": 1}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "json", "source": "/Users/shirhatti/src/Docs/aspnet/data/ef-mvc/intro/samples/cu/appsettings2.json"} -->

````json

   {
     "ConnectionStrings": {
       "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=ContosoUniversity2;Trusted_Connection=True;MultipleActiveResultSets=true"
     },

   ````

This change sets up the project so that the first migration will create a new database. This isn't required for getting started with migrations, but you'll see later why it's a good idea.

Note: As an alternative to changing the database name, you can delete the database. Use **SQL Server Object Explorer** (SSOX) or the `database drop` CLI command:

  <!-- literal_block {"ids": [], "names": [], "highlight_args": {}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "none"} -->

  ````none

     dotnet ef database drop -c SchoolContext
     ````

  The following section explains how to run CLI commands.

## Create an initial migration

Save your changes and build the project. Then open a command window and navigate to the project folder. Here's a quick way to do that:

* In **Solution Explorer**, right-click the project and choose **Open in File Explorer** from the context menu.

  ![Open in File Explorer menu item](migrations/_static/open-in-file-explorer.png)
  ![image](migrations/_static/open-in-file-explorer.png)

* Hold down the Shift key and right-click the project folder in File Explorer, then choose Open command window here from the context menu.

  ![Open command window](migrations/_static/open-command-window.png)
  ![image](migrations/_static/open-command-window.png)

Before you enter a command, stop IIS Express for the site, or you may get an error message: "*cannot access the file ... ContosoUniversity.dll because it is being used by another process.*" To stop the site, find the IIS Express icon in the Windows System Tray, and right-click it, then click **ContosoUniversity > Stop Site**.

After you have stopped IIS Express, enter the following command in the command window:

<!-- literal_block {"ids": [], "names": [], "highlight_args": {}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "text"} -->

````text

   dotnet ef migrations add InitialCreate -c SchoolContext
   ````

You see output like the following in the command window:

<!-- literal_block {"ids": [], "names": [], "highlight_args": {}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "text"} -->

````text

   C:\ContosoUniversity\src\ContosoUniversity>dotnet ef migrations add InitialCreate -c SchoolContext
   Project ContosoUniversity (.NETCoreApp,Version=v1.0) was previously compiled. Skipping compilation.

   Done.

   To undo this action, use 'dotnet ef migrations remove'
   ````

You have to include the `-c SchoolContext` parameter to specify the database context class, because the project has two context classes (the other one is for ASP.NET Identity).

## Examine the Up and Down methods

When you executed the `migrations add` command, EF generated the code that will create the database from scratch. This code is in the *Migrations* folder, in the file named *<timestamp>_InitialCreate.cs*. The `Up` method of the `InitialCreate` class creates the database tables that correspond to the data model entity sets, and the `Down` method deletes them, as shown in the following example.

<!-- literal_block {"ids": [], "names": [], "highlight_args": {"linenostart": 1}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "c#", "source": "/Users/shirhatti/src/Docs/aspnet/data/ef-mvc/intro/samples/cu/Migrations/20160726224716_InitialCreate.cs"} -->

````c#

   public partial class InitialCreate : Migration
   {
       protected override void Up(MigrationBuilder migrationBuilder)
       {
           migrationBuilder.CreateTable(
               name: "Student",
               columns: table => new
               {
                   ID = table.Column<int>(nullable: false)
                       .Annotation("SqlServer:ValueGenerationStrategy", SqlServerValueGenerationStrategy.IdentityColumn),
                   EnrollmentDate = table.Column<DateTime>(nullable: false),
                   FirstMidName = table.Column<string>(nullable: true),
                   LastName = table.Column<string>(nullable: true)
               },
               constraints: table =>
               {
                   table.PrimaryKey("PK_Student", x => x.ID);
               });

           // Additional code not shown
       }

       protected override void Down(MigrationBuilder migrationBuilder)
       {
           migrationBuilder.DropTable(
               name: "Course");
           // Additional code not shown
       }
   }

   ````

Migrations calls the `Up` method to implement the data model changes for a migration. When you enter a command to roll back the update, Migrations calls the `Down` method.

This code is for the initial migration that was created when you entered the `migrations add InitialCreate` command. The migration name parameter ("InitialCreate" in the example) is used for the file name and can be whatever you want. It's best to choose a word or phrase that summarizes what is being done in the migration. For example, you might name a later migration "AddDepartmentTable".

If you created the initial migration when the database already exists, the database creation code is generated but it doesn't have to run because the database already matches the data model. When you deploy the app to another environment where the database doesn't exist yet, this code will run to create your database, so it's a good idea to test it first. That's why you changed the name of the database in the connection string earlier -- so that migrations can create a new one from scratch.

## Examine the data model snapshot

Migrations also creates a "snapshot" of the current database schema in *Migrations/SchoolContextModelSnapshot.cs*. Here's what that code looks like:

<!-- literal_block {"ids": [], "names": [], "highlight_args": {"linenostart": 1}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "c#", "source": "/Users/shirhatti/src/Docs/aspnet/data/ef-mvc/intro/samples/cu/Migrations/SchoolContextModelSnapshot1.cs"} -->

````c#

   [DbContext(typeof(SchoolContext))]
   partial class SchoolContextModelSnapshot : ModelSnapshot
   {
       protected override void BuildModel(ModelBuilder modelBuilder)
       {
           modelBuilder
               .HasAnnotation("ProductVersion", "1.0.0-rtm-21431")
               .HasAnnotation("SqlServer:ValueGenerationStrategy", SqlServerValueGenerationStrategy.IdentityColumn);

           modelBuilder.Entity("ContosoUniversity.Models.Course", b =>
               {
                   b.Property<int>("CourseID");

                   b.Property<int>("Credits");

                   b.Property<int>("DepartmentID");

                   b.Property<string>("Title")
                       .HasAnnotation("MaxLength", 50);

                   b.HasKey("CourseID");

                   b.HasIndex("DepartmentID");

                   b.ToTable("Course");
               });

           // Additional code for Enrollment and Student tables not shown

           modelBuilder.Entity("ContosoUniversity.Models.Enrollment", b =>
               {
                   b.HasOne("ContosoUniversity.Models.Course", "Course")
                       .WithMany("Enrollments")
                       .HasForeignKey("CourseID")
                       .OnDelete(DeleteBehavior.Cascade);

                   b.HasOne("ContosoUniversity.Models.Student", "Student")
                       .WithMany("Enrollments")
                       .HasForeignKey("StudentID")
                       .OnDelete(DeleteBehavior.Cascade);
               });
       }
   }

   ````

Because this code has to reflect the database state after the latest migration,  you can't remove a migration just by deleting the file named  *<timestamp>_<migrationname>.cs*. If you delete that file, the remaining migrations will be out of sync with the database snapshot file. To delete the last migration that you added, use the [dotnet ef migrations remove](https://ef.readthedocs.io/en/latest/miscellaneous/cli/dotnet.html#dotnet-ef-migrations-remove) command.

## Apply the migration to the database

In the command window, enter the following command to create the database and tables in it.

<!-- literal_block {"ids": [], "names": [], "highlight_args": {}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "c#"} -->

````c#

   dotnet ef database update -c SchoolContext
   ````

The output from the command is similar to the `migrations add` command.

<!-- literal_block {"ids": [], "names": [], "highlight_args": {}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "text"} -->

````text

   C:\ContosoUniversity\src\ContosoUniversity>dotnet ef database update -c SchoolContext
   Project ContosoUniversity (.NETCoreApp,Version=v1.0) was previously compiled. Skipping compilation.

   Done.
   ````

Use **SQL Server Object Explorer** to inspect the database as you did in the first tutorial.  You'll notice the addition of an __EFMigrationsHistory table that keeps track of which migrations have been applied to the database. View the data in that table and you'll see one entry for the first migration.

![Migrations history in SSOX](migrations/_static/migrations-table.png)
![image](migrations/_static/migrations-table.png)

Run the application to verify that everything still works the same as before.

![Students Index page](migrations/_static/students-index.png)
![image](migrations/_static/students-index.png)

## Command line interface (CLI) vs. Package Manager Console (PMC)

The EF tooling for managing migrations is available from .NET Core CLI commands or from PowerShell cmdlets in the Visual Studio **Package Manager Console** (PMC) window. In this preview version of the tooling, the CLI commands are more stable than the PMC cmdlets, so this tutorial shows how to use the .NET Core CLI commands.

For more information about the CLI commands, see [.NET Core CLI](https://ef.readthedocs.io/en/latest/miscellaneous/cli/dotnet.html). For information about the PMC commands, see [Package Manager Console (Visual Studio)](https://ef.readthedocs.io/en/latest/miscellaneous/cli/powershell.html).

## Summary

In this tutorial, you've seen how to create and apply your first migration. In the next tutorial, you'll begin looking at more advanced topics by expanding the data model. Along the way you'll create and apply additional migrations.
