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
