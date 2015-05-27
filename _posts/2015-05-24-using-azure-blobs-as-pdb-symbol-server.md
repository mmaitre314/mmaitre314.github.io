---
layout: post
title: Using Azure Blobs as PDB Symbol Server
comments: true
---

Visual Studio uses simple HTTP GET requests to download PDB symbols during debugging, which makes Azure Blobs a great maintenance-free candidate to serve those files. The main trick is to structure the blob paths the way Visual Studio expects.

One way to do that (among others) is to use [SymStore](https://msdn.microsoft.com/en-us/library/windows/desktop/ms681417(v=vs.85).aspx) from the [Windows SDK](https://msdn.microsoft.com/en-US/windows/desktop/bg162891) to create the folder structure locally and then [Azure PowerShell](http://azure.microsoft.com/en-us/documentation/articles/powershell-install-configure/) to upload the files. As an extra bonus SymStore enables symbol-file compression.


{% highlight PowerShell %}
# Find symstore.exe
if (Test-Path "${env:ProgramFiles(x86)}\Windows Kits\8.1\Debuggers\x86\symstore.exe")
{
    $symstorePath = "${env:ProgramFiles(x86)}\Windows Kits\8.1\Debuggers\x86\symstore.exe"
}
elseif (Test-Path "${env:ProgramFiles}\Windows Kits\8.1\Debuggers\x86\symstore.exe")
{
    $symstorePath = "${env:ProgramFiles}\Windows Kits\8.1\Debuggers\x86\symstore.exe"
}
else
{
    throw "symstore.exe not found. Please instead the Windows 8.1 SDK."
}

# Create a temporary folder for SymStore to index the PDBs into
$tempSymbolPath = "${env:TEMP}\Symbols\"
New-Item -ItemType Directory -Path $tempSymbolPath -Force | Out-Null

# Index and compress the PDBs (assumed to be in ${env:TEMP} for the sake of demo)
foreach ($file in Get-ChildItem "${env:TEMP}\*.pdb")
{
    & $symstorePath add /f "$($file.FullName)" /compress /s "$tempSymbolPath" /t "MyNuGet.PackageId" /o
    if ($LASTEXITCODE -ne 0)
    {
        throw "symstore returned $LASTEXITCODE"
    }
}

# Upload the indexed-and-compressed PDBs
$storageAccountName = "mmaitre314"
$storageAccountKey = Get-AzureStorageKey $storageAccountName | %{ $_.Primary }
$context = New-AzureStorageContext -StorageAccountName $storageAccountName -StorageAccountKey $storageAccountKey
foreach ($file in Get-ChildItem "${tempSymbolPath}*.pd_" -Recurse)
{
    $blob = $file.FullName.Substring($tempSymbolPath.Length).Replace("\", "/")
    Set-AzureStorageBlobContent -BlobType Block -Blob $blob -Container "symbols" -File $file.FullName -Context $context -Force
}
{% endhighlight %}

For a full code sample see this NuGet package [publication script](https://github.com/mmaitre314/MediaReader/blob/master/MediaCaptureReader/Package/publish.ps1).

Once a symbol server has been set up, the next step is to extend it to provide source files along symbols. This is left as an exercise to the reader (because I have not done it yet...) but the [GitHub Source Symbol Indexer](http://hamishgraham.net/post/GitHub-Source-Symbol-Indexer.aspx) script provides a good starting point.

*Edit:* the source-server doc pre-dates Git and it shows, but getting the debugger to retrieve source files from GitHub is actually just a matter of injecting a short text file into the PDB using pdbstr.exe from the Windows SDK.

{% highlight yaml %}
SRCSRV: ini ------------------------------------------------
VERSION=2
VERCTRL=http
SRCSRV: variables ------------------------------------------
HTTP_ALIAS=https://raw.githubusercontent.com/mmaitre314/IpCamera/v1.0.1/
HTTP_EXTRACT_TARGET=%HTTP_ALIAS%%var2%
SRCSRVTRG=%HTTP_EXTRACT_TARGET%
SRCSRV: source files ---------------------------------------
c:\users\matthieu\source\repos\ipcamera\ipcamera\ipcamera.shared\debuggerlogger.h*IpCamera/IpCamera.Shared/DebuggerLogger.h
c:\users\matthieu\source\repos\ipcamera\ipcamera\ipcamera.shared\pch.h*IpCamera/IpCamera.Shared/pch.h
c:\users\matthieu\source\repos\ipcamera\ipcamera\ipcamera.shared\cameraserver.cpp*IpCamera/IpCamera.Shared/CameraServer.cpp
c:\users\matthieu\source\repos\ipcamera\ipcamera\ipcamera.shared\connection.h*IpCamera/IpCamera.Shared/Connection.h
c:\users\matthieu\source\repos\ipcamera\ipcamera\ipcamera.shared\cameraserver.h*IpCamera/IpCamera.Shared/CameraServer.h
c:\users\matthieu\source\repos\ipcamera\ipcamera\ipcamera.shared\connection.cpp*IpCamera/IpCamera.Shared/Connection.cpp
c:\users\matthieu\source\repos\ipcamera\ipcamera\ipcamera.shared\debuggerlogger.cpp*IpCamera/IpCamera.Shared/DebuggerLogger.cpp
c:\users\matthieu\source\repos\ipcamera\ipcamera\ipcamera.shared\pch.cpp*IpCamera/IpCamera.Shared/pch.cpp
SRCSRV: end ------------------------------------------------
{% endhighlight %}

Two more links with more detailed background on symbol servers:

- [Sourcepack](https://docs.google.com/document/d/13VM59LEuNps66TK_vITqKd1TPlUtKDaOg9T2dLnFXnE/edit?hl=en_US)
- [MSDN](https://msdn.microsoft.com/en-us/library/windows/hardware/ff540151(v=vs.85).aspx)
