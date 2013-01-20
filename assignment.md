# License management in popular applications	 #

## Intro ##

As a software developer who aims to end up in product development (i.e. the development of applications for general, public use), I know that someday, I will need to create some sort of protection for a non-free application.

Most current paid-for software relies on a so called *license key* or *serial*. More importantly, a lot of applications come in the form of a free trial that can be "fully unlocked" by entering this license key.

My goal, for this assignment, was to have a look at how this protection is implemented in currently available, non-free software, looking at the various ways this may be circumvented (by creating and applying a crack) or defeated (by creating a keygen - Key Generator). The educational value in this whole endeavour is to figure out popular solutions, their relative safety, and ways in which this could be done better.

At the start of this project, I was not aware of the availability and level of functionality of high level CLR decompilers, and as such, went through the processes described in this paper in a much longer winded and complicated manner.

## JetBrains dotCover ##

[JetBrains dotCover](http://www.jetbrains.com/dotcover/) is a .NET unit test runner and code coverage tool that integrates with Visual Studio. JetBrains' set of Visual Studio plugins is widely popular, judging by the number of times one of their various tools is recommended on sites such as [stackoverflow](http://stackoverflow.com).

I chose this piece of software for various reasons. After some research, I could not find an existing keygen for it, anywhere. Not only that, but there is a free, 30-day trial available for it. It is also a fairly small and simple application, so I wouldn't have to grasp the flow through a huge application, but rather, I'd be able to assume that most of the license-key logic would be found in one single place. Last but not least, being part of a suite made me wonder whether the other tools in the suite would have similar protection schemes.

A dotCover license does not come cheap. A single seat for a commercial license runs at €189. A personal license goes for €94. There are free classroom (for trainers and educational institutions) and open source licenses, though.

Other tools available in this suite are comparably expensive. A commercial license for dotTrace Memory + Performance professional edition (memory and performance profilers for .NET applicaitons) will cost you €712+BTW for a single seat. ReSharper, one of the flagship products developed by JetBrains, will lighten your wallet by €332+BTW. The point being, this is professional software, written by people with deep insights into the way .NET applications work, are built, and with plenty of expertise writing professional grade software.

### Time for a little background information ###

What we commonly call .NET applications, are in fact, applications written in one (or more) of the languages written for the *CLR*, leveraging the .NET framework. The *CLR* or *Common Language Runtime* is, much like the *JVM* (*Java Virtual Machine*), a runtime that executes applications by interpreting and JITing (*Just In Time* compilation) so called *CIL*, *Common Intermediate Language*. The CLR is actually Microsoft's  implementation of the CLI (Common Language Infrastructure) standard. Other implementations of the CLI standard include *Shared Source Common Language Infrastructure* - Microsoft's reference implementation released under their Shared Source licensing program, Microsoft Silverlight - an implementation designed for use on the web, and Mono, a well known open source implementation of CLI and accompanying technologies.

This is very comparable to Java bytecode. Not only their pipeline is very similar, their implementation as well: both *CIL* and Java bytecode are an assembler like language, running in a stack-based (vs. the usual register based) virtual machine of runtime, and both are object oriented. One major difference is that the CIL is completely statically typed while the java bytecode instructionset has some support for dynamically typed languages built-in.

Like in the JVM world where Java is not the only language that can be compiled to something the JVM can run, there are a number of languages that can easily be compiled down to something the CLR can run.

Examples of languages that can be compiled down to CIL include C#, Visual Basic .NET, F# and C++/CLI.

### High level .NET decompilers ###

By default, a lot of symbols are retained when code is compiled down to CIL. This makes it possible to reverse engineer the original high-level code based only on the executables. Programs that are capable of not only disassembling the CIL (such as ILDasm) are referred to as decompilers.

For these decompilers, it doesn't matter much in what language the application was originally written in, most can be reversed based solely on the CIL. Some of these decompilers - such as JetBrains' dotPeek and Telerik's JustDecompile - allow to decompile to C# only, while others may also optionally decompile to other languages, such as RedGate's Reflector which is capable of decompiling to VB.Net and F# as well.

### Finding a starting point ###

DotCover doesn't come packaged as a single executable. Instead, it is a bunch of dll's, as well as an executable for command-line use. Not just *a couple* of dll's. No, 193 dll's and 16 executables.

Trying to blindly find and decompile whichever (set of) dll's and/or executables might be handling the license verification is akin to finding a needle in a haystack. So, we'll try a different approach. By entering a bogus license, we can be fairly certain that the license will be checked, and some sort of error message will be displayed. If we find the code responsible for displaying that message, we should be able to backtrack down the application, searching for (and hopefully finding) the root of our problem - the code that checked our bogus license and decided it was, well, bogus.

DotCover shows us a lovely *The license key is invalid* message upon entering a bogus License Key. Let's try to find all the dll's that mention this string somewhere, and hope that this will narrow it down somewhat. Searching for a string within the contents of a file seems like a regular day at work for `strings` and `grep`. Since we're in windows, though, we'll leverage some powershell functionality to replace the grep call, and use a downloaded `strings` executable.

    C:\Program Files (x86)\JetBrains\dotCover\v2.2\Bin> strings *.dll | where {$_ -match "The license key is invalid"}
    .\JetBrains.Platform.dotCover.Shell.dll: The license key is invalid
    .\JetBrains.Platform.dotCover.UI.dll: The license key is invalid

Okay, so it appears in 2 dll's. That's not so bad. Especially not since they both seem to be related to some sort of front-end. The *UI* dll seems like it's more likely to have been responsible for displaying the failure message.

### Digging down ###

We'll start by decompiling `JetBrains.Platform.dotCover.UI.dll` (using JetBrains' own dotPeek) and having a look at the code generating that message. After some digging, we find a class - `JetBrains.UI.Application.License.LicenseInformationControl` - which looks to be the Controller class for the License Information view. Makes sense. In there, we find a method:

    private bool CheckLicense()
    {
      this.myOkButton.Enabled = false;
      if (this.UserName.Length == 0)
      {
        this.myDetailsLabel.Text = "User name cannot be empty";
        this.myDetailsLabel.ForeColor = Color.Red;
        return false;
      }
      else
      {
        string licenseString = this.LicenseString;
        this.myDetailsLabel.Text = string.IsNullOrEmpty(licenseString) ? "License key cannot be empty" : "The license key is invalid";
        if (!string.IsNullOrEmpty(licenseString))
        {
          LicenseData licenseData = this.myDescriptor.LicenseSupport.CreateLicenseData(licenseString, this.UserName, string.Empty);
          LicenseCheckResult licenseCheckResult = licenseData.Check();
          if (licenseCheckResult != LicenseCheckResult.LICENSE_INVALID && this.myUserLicenseRadio.Checked)
            this.SelectEdition(licenseData.GetEdition(this.myDescriptor.LicenseSupport, this.myDescriptor));
          if (licenseCheckResult == LicenseCheckResult.LICENSE_EXPIRED)
          {
            this.myDetailsLabel.ForeColor = Color.Red;
            this.myDetailsLabel.Text = "Your license has expired on " + licenseData.ExpirationDate.ToLongDateString();
            return false;
          }
          else if (licenseCheckResult == LicenseCheckResult.LICENSE_VALID)
          {
            this.myDetailsLabel.ForeColor = SystemColors.ControlText;
            this.myDetailsLabel.Text = "Expiration date: " + (licenseData.ExpirationDate == DateTime.MaxValue ? "never" : licenseData.ExpirationDate.ToLongDateString());
            this.myOkButton.Enabled = true;
            return true;
          }
          else
            this.myDetailsLabel.Text = licenseCheckResult.Message;
        }
        this.myDetailsLabel.ForeColor = Color.Red;
        return false;
      }
    }

Seems fairly straightforward. If a license key has been entered, it first assumes that the key is invalid. It then creates a `LicenseData` instance, calls the `Check()` method on it, which returns a `LicenseCheckResult` instance. By the looks of it, that LicenseCheckResult is sort of a hacky way of simulating a Java enum - a C# enum is not much more than an int, whereas a Java enum is actually an instance of a class that can be statically referred to, with some extra syntactic sugar.

At some point, we'll probably have to have a look at the `LicenseData` class, but first, let's have a look at the `this.myDescriptor.LicenseSupport.CreateLicenseData(..)` line. The parameters make sense - the license string as entered by the user, their username, and a third, `String.Empty` parameter, for the companyname.

The `LicenseData` class appears to be defined in `JetBrains.Application.License.licenseData`, contained within the `JetBrains.Platform.dotCover.Shell.dll` file. We didn't need to do any work figuring that out, JetBrains' dotPeek is actually clever enough to figure that out.

We run into a little problem, here, though: `LicenseData` has 2 constructors, and both of them require a 4th parameter that we didn't explicitely pass to the Factory method above: a `string publicKey` parameter. Presumably, it was injected by the `this.myDescriptor.LicenseSupport.CreateLicenseData` call. `myDescriptor` was of type `IApplicationDescriptor` - an interface. Luckily, dotPeek can search for all derived types, and we end up looking at ... well, nothing much. We can only search in dll's the current dll refers to. But these are "platform" dll's, which are likely supposed to be reusable, and as such, they shouldn't (and aren't) referring to any other dll's. So, we need to find a dll that contains an implementation of `IApplicationDescriptor`.

There's some ways we could do that. We could load all dll's, and re-run the search for derived types. We could do some grep-fu. Or we can just make an educated guess.

`JetBrains.dotCover.ShellBase.dll` sort of jumps out. So we load it, re-run the search for derived types of `IApplicationDescriptor` and stumble upon the `DotCoverBaseApplicationDescriptor`. Although that's still an abstract class, but at least it tells us where we can find an implementation for `ILicenseSupport` so we can see the implementation of the `CreateLicenseData(..)` method: `DotCoverLicenseSupport`.

One thing about the constructor that sort of really jumps out, is the following bit:

    this.myLicensePublicKeys.Add(
      "3127336858484881521162666190662554489729299255697760308701",
      (LicenseData.AcceptLicenseDelegate) (data =>
      {
        DateTime local_0 = data.ContainsSubscription ? data.SubscriptionEndDate : data.GenerationDate.AddYears(1);
        if (!(local_0.AddDays(1.0) <= coverLicenseSupport.ProductBuiltDateUsedForSubscriptionCheck))
          return LicenseCheckResult.LICENSE_VALID;
        return new LicenseCheckResult("Subscription is valid only for {0} builds released before {1}", new object[2]
        {
          (object) descriptor.ProductName,
          (object) local_0.ToShortDateString()
        });
      }));

Presumably, that AcceptLicenseDelegate lambda will be ran at some later point in time, as an extra on top of the regular license checking routine. What's more important here, though, is that public key. The 4th parameter to the constructor of `LicenseData`. And indeed, the `CreateLicenseData(..)` method confirms our hunch, it passes the the key as well as the delegate down to the `LicenseData` constructor.

### LicenseData ###

    public LicenseCheckResult Check()
    {
      if (this.myLicenseCheckResult != null)
        return this.myLicenseCheckResult;
      LicenseCheckResult licenseCheckResult = this.CheckValidity();
      if (licenseCheckResult == LicenseCheckResult.LICENSE_VALID && this.myAcceptLicenseDelegate != null)
        licenseCheckResult = this.myAcceptLicenseDelegate(this);
      this.myLicenseCheckResult = licenseCheckResult;
      return this.myLicenseCheckResult;
    }

Fairly obvious: if we haven't checked yet, check, and if it appears valid, apply the extra check provided through that AcceptLicenseDelegate up there. So, we dig along, and check out the `CheckValidity()` method.

Skimming through it, another redirection follows. The actual checking seems to be taking place in the `LicenseChecker` class. Makes sense.

### LicenseChecker ###

This is what we're looking for. First of all, the constructor that's called in the `CheckValidity` method looks as follows.

    public LicenseChecker(string publickey, string username, string company, string license)
      : this(new BigInteger(publickey, 10), username, company, license)
    {
    }

So, it converts the publickey into a `BigInteger`. However, C#'s built in `BigInteger` class doesn't have a `(string, string)` constructor. A little investigation later, we figure out that it's a C# reimplementation of Java's `BigInteger` class, most likely because previous versions of C# didn't have a `BigInteger` class. However, it doesn't really matter much - a BigInteger is a BigInteger, so let's just regard it as "a really large number" and move on.

    public LicenseChecker(BigInteger n, string username, string company, string license)
    {
      this.myHasLicense = license.Length > 0;
      this.N = n;
      this.myUsername = username.Trim();
      this.myCompany = company;
      foreach (Func<string, byte[]> func in LicenseChecker.StringToByteConvertors)
      {
        try
        {
          this.myCode = new BigInteger(Convert.FromBase64String(license));
        }
        catch (FormatException ex)
        {
          this.myCode = BigInteger.Zero;
          continue;
        }
        this.myCode = this.myCode.modPow(new BigInteger(func(username)) | (BigInteger) 1, this.N);
        if (this.IsChecksumOK)
          break;
      }
    }

So, first of all, it sets `myHasLicense` if the user's license has a length. `N` is set to the public key, `myUsername` to the username, `myCompany` to the company name. Then, it runs through all the `StringToByteConvertors` which are likely meant to turn a string into a `byte[]`. Looking at `LicenseChecker.StringToByteConvertors`, it's just a private static field, initialized to an array existing of only one element: `StringToBytesConverter.Old`.

    public static byte[] Old(string s)
    {
      byte[] numArray = new byte[s.Length];
      for (int index = 0; index < s.Length; ++index)
      {
        int num = (int) s[index];
        numArray[index] = (byte) (num & (int) sbyte.MaxValue);
      }
      return numArray;
    }

So, nothing special. An array of bytes, with each character in the string clipped to the maximum value of a "signed byte", an 8 bit signed integer. 127.

So, let's keep in mind that `func` is now just `StringToBytesConverter.Old`.

Now, we try to parse the entered licensekey by converting it from a base64 encoded string to a byte array, and instantiating a new biginteger from it. So, we have another important piece of information: our license is base64-encoded.

Then, a mathematical transformation is applied to the BigInteger produced above: a modular exponentiation with our license-as-an-int as the base, `func(username) | 1` as the exponent and `N`, the public key, as the modulus. It the result of that operation that will then be used to extract all the information described within the licensekey.

### The License ###

As it turns out, there's a rather large amount of information hidden in that integer:

-   (deprecated) version information
    
    The comments on this method indicate that this is included primarily for compatibility with "OmniaMea" compatibility, a discontinued JetBrains products
-   License type information

    There are apparently a whole bunch of possible license types built in: INVALID, COMMERCIAL, NON_COMMERCIAL, SITE, OPENSOURCE, PERSONAL, ACADEMIC, CLASSROOM, FLOATING
-   Date the license was generated
    
    This, as well as other dates, is embedded using bitmasks and shifting to fit the date into a 16bit value, as a concatenation of the day, month and year - 2003, with 2 special values: `0xFF` which maps to `DateTime.MaxValue` and `0x00` which maps to `DateTime.MinValue`.
-   Date the license will expire
-   Date the license subscription expires
-   Product version

    This one has a fairly simple mapping: 2.2 is encoded as 2002
-   Customer ID (only if applicable)

    That is to say, if the license entered by the user is prepended by a number and a dash, for example "123-", then the Customer ID must be equal to that number.
-   Edition (presumably for products that come in different "flavours" such as ReFactor which comes in 3 flavours: Full, C# and VB.Net)
-   A checksum

    This checksum is compared to a sort of "hashing" operation executed on the username as well as the company name. This is really what ties a license to a certain user.

Each of these pieces of information is allotted some space in the license. The layout is as follows:

- 16 bits: Subscription end date
- 8 bits: *reserved*
- 8 bits: Edition
- 32 bits: Customer ID
- 16 bits: Product version 
- 16 bits: Expiration date
- 16 bits: Generation date
- 8 bits: License type
- 32 bits: Checksum

Using all of that information, it is fairly simple to create a valid "unencrypted" license. Ignoring the subscription end-date for a while, as I'm fairly sure it's not needed for dotCover licenses, we end up with something akin to the following:

    var myCode = new BigInteger();
    myCode |= edition;
    myCode = myCode << 0x30;
    myCode |= keyDef.ProductVersion.ToInt();
    myCode = myCode << 0x10;
    myCode |= 0xffff; // expiration date
    myCode = myCode << 0x10;
    myCode |= 0x408A; // generation date
    myCode = myCode << 0x08;
    myCode |= (int)licenseType;
    myCode = myCode << 0x20;
    myCode |= CalculateUserHash(username, String.Empty);

### Reversing the encryption ###

If we can't reverse the modular exponentiation operation, we're done and over with. This is clearly a sort of implementation of RSA. We already saw the public key elsewhere, and lo and behold, it's a really short key. One of the reasons RSA *works*, is because it's hard to find the private key given only the public key. That because it's hard to factorize large numbers. This, however, is a fairly small number. So let's see if we can factorize it on the internet. We find a website, [The Number Empire](www.numberempire.com), which has [a tool](http://www.numberempire.com/numberfactorizer.php) for factorizing numbers up to 60 decimal digits in length. We feed it the public key we found earlier, and find that `3127336858484881521162666190662554489729299255697760308701 = 43494238288651199977873075199 * 71902325032805244668141336099`

Now finding the private key becomes feasible. With P and Q the factors of N:

    var userHash = new BigInteger(func(username));
    var exp = (userHash | 1).ModInverse((P - 1) * (Q - 1));

The ModInverse method show here applies [the Extended Euclidean Algorithm](http://en.wikipedia.org/wiki/Extended_Euclidean_algorithm) and returns the modular multiplicative inverse of `(userHash | 1)`, the value the keychecking uses as the encryption exponent

Now we can turn our myCode into an actual license in decimal representation:

    var license = myCode.ModPow(exp, keyDef.PubKey);

By this point, writing a key generator has become trivial. So let's do one better.

## The JetBrains .NET Tools ##

JetBrains has 5 products in its .NET tools suite. One of them is free, but the others are all "for money". As a matter of fact, they all go for a fair bit of money.

Due to all the references to "platform" and the lengths the developers have clearly gone to, in order to make the whole licensing system as reusable as possible, it would only make sense if the other products would use the same system for checking licenses.

And clearly, that's the case. They only differ in their Product Version and - or course - their public key. Creating a generic keygen that would work on each and every one of the JetBrains .NET Tools is now not only within reach, but exceedingly trivial. And so I did.