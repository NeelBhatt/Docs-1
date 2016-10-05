---
uid: security/data-protection/consumer-apis/password-hashing
---
# Password Hashing

The data protection code base includes a package *Microsoft.AspNetCore.Cryptography.KeyDerivation* which contains cryptographic key derivation functions. This package is a standalone component and has no dependencies on the rest of the data protection system. It can be used completely independently. The source exists alongside the data protection code base as a convenience.

The package currently offers a method `KeyDerivation.Pbkdf2` which allows hashing a password using the [PBKDF2 algorithm](https://tools.ietf.org/html/rfc2898#section-5.2). This API is very similar to the .NET Framework's existing [Rfc2898DeriveBytes type](https://msdn.microsoft.com/en-us/library/System.Security.Cryptography.Rfc2898DeriveBytes(v=vs.110).aspx), but there are three important distinctions:

1. The `KeyDerivation.Pbkdf2` method supports consuming multiple PRFs (currently `HMACSHA1`, `HMACSHA256`, and `HMACSHA512`), whereas the `Rfc2898DeriveBytes` type only supports `HMACSHA1`.

2. The `KeyDerivation.Pbkdf2` method detects the current operating system and attempts to choose the most optimized implementation of the routine, providing much better performance in certain cases. (On Windows 8, it offers around 10x the throughput of `Rfc2898DeriveBytes`.)

3. The `KeyDerivation.Pbkdf2` method requires the caller to specify all parameters (salt, PRF, and iteration count). The `Rfc2898DeriveBytes` type provides default values for these.

[!code-none[Main](password-hashing/samples/passwordhasher.cs)]

````none

   using System;
   using System.Security.Cryptography;
   using Microsoft.AspNetCore.Cryptography.KeyDerivation;
    
   public class Program
   {
       public static void Main(string[] args)
       {
           Console.Write("Enter a password: ");
           string password = Console.ReadLine();
    
           // generate a 128-bit salt using a secure PRNG
           byte[] salt = new byte[128 / 8];
           using (var rng = RandomNumberGenerator.Create())
           {
               rng.GetBytes(salt);
           }
           Console.WriteLine($"Salt: {Convert.ToBase64String(salt)}");
    
           // derive a 256-bit subkey (use HMACSHA1 with 10,000 iterations)
           string hashed = Convert.ToBase64String(KeyDerivation.Pbkdf2(
               password: password,
               salt: salt,
               prf: KeyDerivationPrf.HMACSHA1,
               iterationCount: 10000,
               numBytesRequested: 256 / 8));
           Console.WriteLine($"Hashed: {hashed}");
       }
   }
    
   /*
    * SAMPLE OUTPUT
    *
    * Enter a password: Xtw9NMgx
    * Salt: NZsP6NnmfBuYeJrrAKNuVQ==
    * Hashed: /OOoOer10+tGwTRDTrQSoeCxVTFr6dtYly7d0cPxIak=
    */
   ````

See the source code for ASP.NET Core Identity's `PasswordHasher` type for a real-world use case.
