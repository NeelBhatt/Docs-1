---
uid: security/data-protection/consumer-apis/dangerous-unprotect
---
<a name=data-protection-consumer-apis-dangerous-unprotect></a>

# Unprotecting payloads whose keys have been revoked

The ASP.NET Core data protection APIs are not primarily intended for indefinite persistence of confidential payloads. Other technologies like [Windows CNG DPAPI](https://msdn.microsoft.com/en-us/library/windows/desktop/hh706794%28v=vs.85%29.aspx) and [Azure Rights Management](https://technet.microsoft.com/en-us/library/jj585024.aspx) are more suited to the scenario of indefinite storage, and they have correspondingly strong key management capabilities. That said, there is nothing prohibiting a developer from using the ASP.NET Core data protection APIs for long-term protection of confidential data. Keys are never removed from the key ring, so IDataProtector.Unprotect can always recover existing payloads as long as the keys are available and valid.

However, an issue arises when the developer tries to unprotect data that has been protected with a revoked key, as IDataProtector.Unprotect will throw an exception in this case. This might be fine for short-lived or transient payloads (like authentication tokens), as these kinds of payloads can easily be recreated by the system, and at worst the site visitor might be required to log in again. But for persisted payloads, having Unprotect throw could lead to unacceptable data loss.

## IPersistedDataProtector

To support the scenario of allowing payloads to be unprotected even in the face of revoked keys, the data protection system contains an IPersistedDataProtector type. To get an instance of IPersistedDataProtector, simply get an instance of IDataProtector in the normal fashion and try casting the IDataProtector to IPersistedDataProtector.

> [!NOTE]
> Not all IDataProtector instances can be cast to IPersistedDataProtector. Developers should use the C# as operator or similar to avoid runtime exceptions caused by invalid casts, and they should be prepared to handle the failure case appropriately.

IPersistedDataProtector exposes the following API surface:

````csharp

   DangerousUnprotect(byte[] protectedData, bool ignoreRevocationErrors,
     out bool requiresMigration, out bool wasRevoked) : byte[]
   ````

This API takes the protected payload (as a byte array) and returns the unprotected payload. There is no string-based overload. The two out parameters are as follows.

* requiresMigration: will be set to true if the key used to protect this payload is no longer the active default key, e.g., the key used to protect this payload is old and a key rolling operation has since taken place. The caller may wish to consider reprotecting the payload depending on his business needs.

* wasRevoked: will be set to true if the key used to protect this payload was revoked.

>[!WARNING]
> Exercise extreme caution when passing ignoreRevocationErrors: true to the DangerousUnprotect method. If after calling this method the wasRevoked value is true, then the key used to protect this payload was revoked, and the payload's authenticity should be treated as suspect. In this case only continue operating on the unprotected payload if you have some separate assurance that it is authentic, e.g. that it's coming from a secure database rather than being sent by an untrusted web client.

<!-- literal_block {"xml:space": "preserve", "source": "security/data-protection/consumer-apis/dangerous-unprotect/samples/dangerous-unprotect.cs", "ids": [], "linenos": true, "language": "none", "highlight_args": {"linenostart": 1}} -->

````none

   using System;
   using System.IO;
   using System.Text;
   using Microsoft.AspNetCore.DataProtection;
   using Microsoft.AspNetCore.DataProtection.KeyManagement;
   using Microsoft.Extensions.DependencyInjection;

   public class Program
   {
       public static void Main(string[] args)
       {
           var serviceCollection = new ServiceCollection();
           serviceCollection.AddDataProtection()
               // point at a specific folder and use DPAPI to encrypt keys
               .PersistKeysToFileSystem(new DirectoryInfo(@"c:\temp-keys"))
               .ProtectKeysWithDpapi();
           var services = serviceCollection.BuildServiceProvider();

           // get a protector and perform a protect operation
           var protector = services.GetDataProtector("Sample.DangerousUnprotect");
           Console.Write("Input: ");
           byte[] input = Encoding.UTF8.GetBytes(Console.ReadLine());
           var protectedData = protector.Protect(input);
           Console.WriteLine($"Protected payload: {Convert.ToBase64String(protectedData)}");

           // demonstrate that the payload round-trips properly
           var roundTripped = protector.Unprotect(protectedData);
           Console.WriteLine($"Round-tripped payload: {Encoding.UTF8.GetString(roundTripped)}");

           // get a reference to the key manager and revoke all keys in the key ring
           var keyManager = services.GetService<IKeyManager>();
           Console.WriteLine("Revoking all keys in the key ring...");
           keyManager.RevokeAllKeys(DateTimeOffset.Now, "Sample revocation.");

           // try calling Protect - this should throw
           Console.WriteLine("Calling Unprotect...");
           try
           {
               var unprotectedPayload = protector.Unprotect(protectedData);
               Console.WriteLine($"Unprotected payload: {Encoding.UTF8.GetString(unprotectedPayload)}");
           }
           catch (Exception ex)
           {
               Console.WriteLine($"{ex.GetType().Name}: {ex.Message}");
           }

           // try calling DangerousUnprotect
           Console.WriteLine("Calling DangerousUnprotect...");
           try
           {
               IPersistedDataProtector persistedProtector = protector as IPersistedDataProtector;
               if (persistedProtector == null)
               {
                   throw new Exception("Can't call DangerousUnprotect.");
               }

               bool requiresMigration, wasRevoked;
               var unprotectedPayload = persistedProtector.DangerousUnprotect(
                   protectedData: protectedData,
                   ignoreRevocationErrors: true,
                   requiresMigration: out requiresMigration,
                   wasRevoked: out wasRevoked);
               Console.WriteLine($"Unprotected payload: {Encoding.UTF8.GetString(unprotectedPayload)}");
               Console.WriteLine($"Requires migration = {requiresMigration}, was revoked = {wasRevoked}");
           }
           catch (Exception ex)
           {
               Console.WriteLine($"{ex.GetType().Name}: {ex.Message}");
           }
       }
   }

   /*
    * SAMPLE OUTPUT
    *
    * Input: Hello!
    * Protected payload: CfDJ8LHIzUCX1ZVBn2BZ...
    * Round-tripped payload: Hello!
    * Revoking all keys in the key ring...
    * Calling Unprotect...
    * CryptographicException: The key {...} has been revoked.
    * Calling DangerousUnprotect...
    * Unprotected payload: Hello!
    * Requires migration = True, was revoked = True
    */
   ````
