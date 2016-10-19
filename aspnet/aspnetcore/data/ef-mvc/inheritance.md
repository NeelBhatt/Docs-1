---
title: Inheritance
author: rick-anderson
ms.author: riande
manager: wpickett
ms.date: 42657
ms.topic: article
ms.assetid: 41dc0db7-6f17-453e-aba6-633430609c74
ms.prod: aspnet-core
uid: data/ef-mvc/inheritance
---
# Inheritance

The Contoso University sample web application demonstrates how to create ASP.NET Core 1.0 MVC web applications using Entity Framework Core 1.0 and Visual Studio 2015. For information about the tutorial series, see [the first tutorial in the series](intro.md).

In the previous tutorial you handled concurrency exceptions. This tutorial will show you how to implement inheritance in the data model.

In object-oriented programming, you can use inheritance to facilitate code reuse. In this tutorial, you'll change the `Instructor` and `Student` classes so that they derive from a `Person` base class which contains properties such as `LastName` that are common to both instructors and students. You won't add or change any web pages, but you'll change some of the code and those changes will be automatically reflected in the database.

## Options for mapping inheritance to database tables

The `Instructor` and `Student` classes in the School data model have several properties that are identical:

![Student and Instructor classes](inheritance/_static/no-inheritance.png)
![image](inheritance/_static/no-inheritance.png)

Suppose you want to eliminate the redundant code for the properties that are shared by the `Instructor` and `Student` entities. Or you want to write a service that can format names without caring whether the name came from an instructor or a student. You could create a `Person` base class that contains only those shared properties, then make the `Instructor` and `Student` classes inherit from that base class, as shown in the following illustration:

![Student and Instructor classes deriving from Person class](inheritance/_static/inheritance.png)
![image](inheritance/_static/inheritance.png)

There are several ways this inheritance structure could be represented in the database. You could have a Person table that includes information about both students and instructors in a single table. Some of the columns could apply only to instructors (HireDate), some only to students (EnrollmentDate), some to both (LastName, FirstName). Typically, you'd have a discriminator column to indicate which type each row represents. For example, the discriminator column might have "Instructor" for instructors and "Student" for students.

![Table-per-hierarchy example](inheritance/_static/tph.png)
![image](inheritance/_static/tph.png)

This pattern of generating an entity inheritance structure from a single database table is called table-per-hierarchy (TPH) inheritance.

An alternative is to make the database look more like the inheritance structure. For example, you could have only the name fields in the Person table and have separate Instructor and Student tables with the date fields.

![Table-per-type inheritance](inheritance/_static/tpt.png)
![image](inheritance/_static/tpt.png)

This pattern of making a database table for each entity class is called table per type (TPT) inheritance.

Yet another option is to map all non-abstract types to individual tables. All properties of a class, including inherited properties, map to columns of the corresponding table. This pattern is called Table-per-Concrete Class (TPC) inheritance. If you implemented TPC inheritance for the Person, Student, and Instructor classes as shown earlier, the Student and Instructor tables would look no different after implementing inheritance than they did before.

TPC and TPH inheritance patterns generally deliver better performance than TPT inheritance patterns, because TPT patterns can result in complex join queries.

This tutorial demonstrates how to implement TPH inheritance. TPH is the only inheritance pattern that the Entity Framework Core supports.  What you'll do is create a `Person` class, change the `Instructor` and `Student` classes to derive from `Person`, add the new class to the `DbContext`, and create a migration.

## Create the Person class

In the Models folder, create Person.cs and replace the template code with the following code:

<!-- literal_block {"ids": [], "names": [], "highlight_args": {"linenostart": 1}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "c#", "source": "/Users/shirhatti/src/Docs/aspnet/data/ef-mvc/intro/samples/cu/Models/Person.cs"} -->

````c#

   using System.ComponentModel.DataAnnotations;
   using System.ComponentModel.DataAnnotations.Schema;

   namespace ContosoUniversity.Models
   {
       public abstract class Person
       {
           public int ID { get; set; }

           [Required]
           [StringLength(50)]
           [Display(Name = "Last Name")]
           public string LastName { get; set; }
           [Required]
           [StringLength(50, ErrorMessage = "First name cannot be longer than 50 characters.")]
           [Column("FirstName")]
           [Display(Name = "First Name")]
           public string FirstMidName { get; set; }

           [Display(Name = "Full Name")]
           public string FullName
           {
               get
               {
                   return LastName + ", " + FirstMidName;
               }
           }
       }
   }
   ````

## Make Student and Instructor classes inherit from Person

In *Instructor.cs*, derive the Instructor class from the Person class and remove the key and name fields. The code will look like the following example:

<!-- literal_block {"ids": [], "names": [], "highlight_args": {"hl_lines": [8], "linenostart": 1}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "c#", "source": "/Users/shirhatti/src/Docs/aspnet/data/ef-mvc/intro/samples/cu/Models/Instructor.cs"} -->

````c#

   using System;
   using System.Collections.Generic;
   using System.ComponentModel.DataAnnotations;
   using System.ComponentModel.DataAnnotations.Schema;

   namespace ContosoUniversity.Models
   {
       public class Instructor : Person
       {
           [DataType(DataType.Date)]
           [DisplayFormat(DataFormatString = "{0:yyyy-MM-dd}", ApplyFormatInEditMode = true)]
           [Display(Name = "Hire Date")]
           public DateTime HireDate { get; set; }

           public ICollection<CourseAssignment> Courses { get; set; }
           public OfficeAssignment OfficeAssignment { get; set; }
       }
   }

   ````

Make the same changes in *Student.cs*.

<!-- literal_block {"ids": [], "names": [], "highlight_args": {"hl_lines": [8], "linenostart": 1}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "c#", "source": "/Users/shirhatti/src/Docs/aspnet/data/ef-mvc/intro/samples/cu/Models/Student.cs"} -->

````c#

   using System;
   using System.Collections.Generic;
   using System.ComponentModel.DataAnnotations;
   using System.ComponentModel.DataAnnotations.Schema;

   namespace ContosoUniversity.Models
   {
       public class Student : Person
       {
           [DataType(DataType.Date)]
           [DisplayFormat(DataFormatString = "{0:yyyy-MM-dd}", ApplyFormatInEditMode = true)]
           [Display(Name = "Enrollment Date")]
           public DateTime EnrollmentDate { get; set; }


           public ICollection<Enrollment> Enrollments { get; set; }
       }
   }

   ````

## Add the Person entity type to the data model

Add the Person entity type to *SchoolContext.cs*. The new lines are highlighted.

<!-- literal_block {"ids": [], "names": [], "highlight_args": {"hl_lines": [19, 30], "linenostart": 1}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "c#", "source": "/Users/shirhatti/src/Docs/aspnet/data/ef-mvc/intro/samples/cu/Data/SchoolContext.cs"} -->

````c#

   using ContosoUniversity.Models;
   using Microsoft.EntityFrameworkCore;

   namespace ContosoUniversity.Data
   {
       public class SchoolContext : DbContext
       {
           public SchoolContext(DbContextOptions<SchoolContext> options) : base(options)
           {
           }

           public DbSet<Course> Courses { get; set; }
           public DbSet<Enrollment> Enrollments { get; set; }
           public DbSet<Student> Students { get; set; }
           public DbSet<Department> Departments { get; set; }
           public DbSet<Instructor> Instructors { get; set; }
           public DbSet<OfficeAssignment> OfficeAssignments { get; set; }
           public DbSet<CourseAssignment> CourseAssignments { get; set; }
           public DbSet<Person> People { get; set; }

           protected override void OnModelCreating(ModelBuilder modelBuilder)
           {
               modelBuilder.Entity<Course>().ToTable("Course");
               modelBuilder.Entity<Enrollment>().ToTable("Enrollment");
               modelBuilder.Entity<Student>().ToTable("Student");
               modelBuilder.Entity<Department>().ToTable("Department");
               modelBuilder.Entity<Instructor>().ToTable("Instructor");
               modelBuilder.Entity<OfficeAssignment>().ToTable("OfficeAssignment");
               modelBuilder.Entity<CourseAssignment>().ToTable("CourseAssignment");
               modelBuilder.Entity<Person>().ToTable("Person");

               modelBuilder.Entity<CourseAssignment>()
                   .HasKey(c => new { c.CourseID, c.InstructorID });
           }
       }
   }

   ````

This is all that the Entity Framework needs in order to configure table-per-hierarchy inheritance. As you'll see, when the database is updated, it will have a Person table in place of the Student and Instructor tables.

## Create and customize migration code

Save your changes and build the project. Then open the command window in the project folder and enter the following command:

<!-- literal_block {"ids": [], "names": [], "highlight_args": {}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "text"} -->

````text

   dotnet ef migrations add Inheritance -c SchoolContext
   ````

Run the `database update` command:.

<!-- literal_block {"ids": [], "names": [], "highlight_args": {}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "text"} -->

````text

   dotnet ef database update -c SchoolContext
   ````

The command will fail at this point because you have existing data that migrations doesn't know how to handle. You get an error message like the following one:

   The ALTER TABLE statement conflicted with the FOREIGN KEY constraint "FK_CourseAssignment_Person_InstructorID". The conflict occurred in database "ContosoUniversity09133", table "dbo.Person", column 'ID'.

Open *Migrations<timestamp>_Inheritance.cs* and replace the `Up` method with the following code:

<!-- literal_block {"ids": [], "names": [], "highlight_args": {"linenostart": 1}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "c#", "source": "/Users/shirhatti/src/Docs/aspnet/data/ef-mvc/intro/samples/cu/Migrations/20160817215858_Inheritance.cs"} -->

````c#

   protected override void Up(MigrationBuilder migrationBuilder)
   {
       migrationBuilder.DropForeignKey(
           name: "FK_CourseAssignment_Instructor_InstructorID",
           table: "CourseAssignment");

       migrationBuilder.DropForeignKey(
           name: "FK_Department_Instructor_InstructorID",
           table: "Department");

       migrationBuilder.DropForeignKey(
           name: "FK_Enrollment_Student_StudentID",
           table: "Enrollment");

       migrationBuilder.DropIndex(name: "IX_Enrollment_StudentID", table: "Enrollment");

       migrationBuilder.RenameTable(name: "Instructor", newName: "Person");
       migrationBuilder.AddColumn<DateTime>(name: "EnrollmentDate", table: "Person", nullable: true);
       migrationBuilder.AddColumn<string>(name: "Discriminator", table: "Person", nullable: false, maxLength: 128, defaultValue: "Instructor");
       migrationBuilder.AlterColumn<DateTime>(name: "HireDate", table: "Person", nullable: true);
       migrationBuilder.AddColumn<int>(name: "OldId", table: "Person", nullable: true);

       // Copy existing Student data into new Person table.
       migrationBuilder.Sql("INSERT INTO dbo.Person (LastName, FirstName, HireDate, EnrollmentDate, Discriminator, OldId) SELECT LastName, FirstName, null AS HireDate, EnrollmentDate, 'Student' AS Discriminator, ID AS OldId FROM dbo.Student");
       // Fix up existing relationships to match new PK's.
       migrationBuilder.Sql("UPDATE dbo.Enrollment SET StudentId = (SELECT ID FROM dbo.Person WHERE OldId = Enrollment.StudentId AND Discriminator = 'Student')");

       // Remove temporary key
       migrationBuilder.DropColumn(name: "OldID", table: "Person");

       migrationBuilder.DropTable(
           name: "Student");

       migrationBuilder.AddForeignKey(
           name: "FK_Enrollment_Person_StudentID",
           table: "Enrollment",
           column: "StudentID",
           principalTable: "Person",
           principalColumn: "ID",
           onDelete: ReferentialAction.Cascade);

       migrationBuilder.CreateIndex(
            name: "IX_Enrollment_StudentID",
            table: "Enrollment",
            column: "StudentID");
   }

   ````

This code takes care of the following database update tasks:

* Removes foreign key constraints and indexes that point to the Student table.

* Renames the Instructor table as Person and makes changes needed for it to store Student data:

* Adds nullable EnrollmentDate for students.

* Adds Discriminator column to indicate whether a row is for a student or an instructor.

* Makes HireDate nullable since student rows won't have hire dates.

* Adds a temporary field that will be used to update foreign keys that point to students. When you copy students into the Person table they'll get new primary key values.

* Copies data from the Student table into the Person table. This causes students to get assigned new primary key values.

* Fixes foreign key values that point to students.

* Re-creates foreign key constraints and indexes, now pointing them to the Person table.

(If you had used GUID instead of integer as the primary key type, the student primary key values wouldn't have to change, and several of these steps could have been omitted.)

Run the `database update` command again:

<!-- literal_block {"ids": [], "names": [], "highlight_args": {}, "backrefs": [], "dupnames": [], "linenos": false, "classes": [], "xml:space": "preserve", "language": "text"} -->

````text

   dotnet ef database update -c SchoolContext
   ````

(In a production system you would make corresponding changes to the `Down` method in case you ever had to use that to go back to the previous database version. For this tutorial you won't be using the `Down` method.)

Note: It's possible to get other errors when making schema changes in a database that has existing data. If you get migration errors that you can't resolve, you can either change the database name in the connection string or delete the database. With a new database, there is no data to migrate, and the update-database command is more likely to complete without errors. To delete the database, use SSOX or run the `database drop` CLI command.

## Test with inheritance implemented

Run the site and try various pages. Everything works the same as it did before.

In **SQL Server Object Explorer**, expand **Data Connections/SchoolContext** and then **Tables**, and you see that the Student and Instructor tables have been replaced by a Person table. Open the Person table designer and you see that it has all of the columns that used to be in the Student and Instructor tables.

![Person table in SSOX](inheritance/_static/ssox-person-table.png)
![image](inheritance/_static/ssox-person-table.png)

Right-click the Person table, and then click **Show Table Data** to see the discriminator column.

![Person table in SSOX - table data](inheritance/_static/ssox-person-data.png)
![image](inheritance/_static/ssox-person-data.png)

## Summary

You've implemented table-per-hierarchy inheritance for the `Person`, `Student`, and `Instructor` classes. For more information about inheritance in Entity Framework Core, see [Inheritance](https://ef.readthedocs.io/en/latest/modeling/inheritance.html). In the next tutorial you'll see how to handle a variety of relatively advanced Entity Framework scenarios.
