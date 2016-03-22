---
layout: post
title: Publishing C# cmdlets to the PowerShell Gallery
comments: true
---

[PowerShellGet](https://technet.microsoft.com/library/dn807169.aspx) and the [PowerShell Gallery](https://www.powershellgallery.com/) are the equivalent of NuGet/NPM/Bower package managers for PowerShell. They come handy to share scripts and modules.

Installing a module, like [this demo one](https://www.powershellgallery.com/packages/PowerShellGet-Test-Binary-Module/) for instance, is a one-liner:
 
{% highlight PowerShell %}
Install-Module -Name PowerShellGet-Test-Binary-Module -Scope CurrentUser
{% endhighlight %}
 
Creating and publishing a new module is also relatively straightforward, although not exceptionally well documented.

A simple `Write-HelloWorld` cmdlet consists in a class deriving from `Cmdlet` that overrides `BeginProcessing` to output a string:

{% highlight csharp %}
using System.Management.Automation;

[Cmdlet(VerbsCommunications.Write, "HelloWorld")]
public class HelloWorldCmdlet : Cmdlet
{
    protected override void BeginProcessing()
    {
        WriteObject("Hello, World!");
    }
}
{% endhighlight %}

One trick here is finding the `System.Management.Automation` reference. On my machine (Windows 10 x64) it could be found at `c:\Program Files (x86)\Reference Assemblies\Microsoft\WindowsPowerShell\3.0\System.Management.Automation.dll`.

Once compiled, the DLL that can be loaded in PowerShell using `Import-Module`.

To make it a publishable module requires adding a .psd1 PowerShell manifest akin to NuGet's .nuspec XML manifest:

{% highlight PowerShell %}
@{
    RootModule = 'PowerShellGet-Test-Binary-Module.dll'
    ModuleVersion = '1.0.0.3'
    CmdletsToExport = '*'
    GUID = '95ee4cf1-d508-45a8-9680-203b71453f98'
    DotNetFrameworkVersion = '4.5.1'
    Author = 'Matthieu Maitre'
    Description = 'Example of PowerShellGet Binary Module'
    CompanyName = 'None'
    Copyright = '(c) 2016 Matthieu Maitre. All rights reserved.'
    PrivateData = @{
        PSData = @{
            ProjectUri = 'https://github.com/mmaitre314/PowerShellGet-Test-Binary-Module'
            LicenseUri = 'https://github.com/mmaitre314/PowerShellGet-Test-Binary-Module/blob/master/LICENSE'
            ReleaseNotes = ''
        }
    }
}
{% endhighlight %}

Both the DLL and .psd1 file need to be placed in a folder whose name is the package name. Once this is done, the package gets uploaded to the gallery using

{% highlight PowerShell %}
Publish-Module -Path .\PowerShellGet-Test-Binary-Module -Repository PSGallery -NuGetApiKey "xxx"
{% endhighlight %}

where the API Key comes from [https://powershellgallery.com/account](https://powershellgallery.com/account).

That is it. For the full source code and some unit-testing tricks see [this repo](https://github.com/mmaitre314/PowerShellGet-Test-Binary-Module) on GitHub.