---
uid: security/data-protection/using-data-protection
---
<a name=security-data-protection-getting-started></a>

# Getting Started with the Data Protection APIs

At its simplest protecting data consists of the following steps:

1. Create a data protector from a data protection provider.

2. Call the Protect method with the data you want to protect.

3. Call the Unprotect method with the data you want to turn back into plain text.

Most frameworks such as ASP.NET or SignalR already configure the data protection system and add it to a service container you access via dependency injection. The following sample demonstrates configuring a service container for dependency injection and registering the data protection stack, receiving the data protection provider via DI, creating a protector and protecting then unprotecting data

<!-- literal_block {"xml:space": "preserve", "source": "security/data-protection/using-data-protection/samples/protectunprotect.cs", "ids": [], "linenos": true, "language": "none", "highlight_args": {"hl_lines": [26, 34, 35, 36, 37, 38, 39, 40], "linenostart": 1}} -->

````none

   using System;
   using Microsoft.AspNetCore.DataProtection;
   using Microsoft.Extensions.DependencyInjection;

   public class Program
   {
       public static void Main(string[] args)
       {
           // add data protection services
           var serviceCollection = new ServiceCollection();
           serviceCollection.AddDataProtection();
           var services = serviceCollection.BuildServiceProvider();

           // create an instance of MyClass using the service provider
           var instance = ActivatorUtilities.CreateInstance<MyClass>(services);
           instance.RunSample();
       }

       public class MyClass
       {
           IDataProtector _protector;

           // the 'provider' parameter is provided by DI
           public MyClass(IDataProtectionProvider provider)
           {
               _protector = provider.CreateProtector("Contoso.MyClass.v1");
           }

           public void RunSample()
           {
               Console.Write("Enter input: ");
               string input = Console.ReadLine();

               // protect the payload
               string protectedPayload = _protector.Protect(input);
               Console.WriteLine($"Protect returned: {protectedPayload}");

               // unprotect the payload
               string unprotectedPayload = _protector.Unprotect(protectedPayload);
               Console.WriteLine($"Unprotect returned: {unprotectedPayload}");
           }
       }
   }

   /*
    * SAMPLE OUTPUT
    *
    * Enter input: Hello world!
    * Protect returned: CfDJ8ICcgQwZZhlAlTZT...OdfH66i1PnGmpCR5e441xQ
    * Unprotect returned: Hello world!
    */

   ````

When you create a protector you must provide one or more [Purpose Strings](consumer-apis/purpose-strings.md). A purpose string provides isolation between consumers, for example a protector created with a purpose string of "green" would not be able to unprotect data provided by a protector with a purpose of "purple".

>[!TIP]
> Instances of IDataProtectionProvider and IDataProtector are thread-safe for multiple callers. It is intended that once a component gets a reference to an IDataProtector via a call to CreateProtector, it will use that reference for multiple calls to Protect and Unprotect.A call to Unprotect will throw CryptographicException if the protected payload cannot be verified or deciphered. Some components may wish to ignore errors during unprotect operations; a component which reads authentication cookies might handle this error and treat the request as if it had no cookie at all rather than fail the request outright. Components which want this behavior should specifically catch CryptographicException instead of swallowing all exceptions.
