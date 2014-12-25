---
layout: post
title: "Secure String Code Generator: A Breakdown"
comments: true
---

All in all the [code generator](http://mmaitre314.github.io/SecureStringCodeGen/) creating C# classes out of XML config files and environment variables is a small piece of code. Writing it however was more painful than I expected: it takes quite some tricks to get MSBuild and PowerShell to work, resulting in way too many hours spent scouring the web. This blog goes through those tricks to make them a little less obscure.

MSBuild Target
---

The first part of the code is an MSBuild Target which NuGet injects into the VS project during package install. The target -- called CreateSettingsClass -- runs a PowerShell script right before compilation for each file with an .stx extension in the project folder.

{% highlight xml %}
<ItemGroup>
    <SettingsTemplateFiles Include="*.stx" />
</ItemGroup>

<PropertyGroup>
    <CompileDependsOn>$(CompileDependsOn);CreateSettingsClass</CompileDependsOn>
</PropertyGroup>

<Target Name="CreateSettingsClass"
      BeforeTargets="ResolveAssemblyReferences"
      Inputs="@(SettingsTemplateFiles)"
      Outputs="@(SettingsTemplateFiles -> '$(IntermediateOutputPath)%(Filename).stx.cs')"
      >
    <ItemGroup>
      <Compile Include="$(IntermediateOutputPath)%(SettingsTemplateFiles.Filename).stx.cs"/>
    </ItemGroup>
    <Exec Command="powershell.exe –NonInteractive –ExecutionPolicy Unrestricted -Command &quot;&amp; { &amp;&apos;$(MSBuildThisFileDirectory)New-SettingsClass.ps1&apos; &apos;%(SettingsTemplateFiles.FullPath)&apos; &apos;$(MSBuildProjectDirectory)\$(IntermediateOutputPath)%(SettingsTemplateFiles.Filename).stx.cs&apos; $(RootNamespace) %(SettingsTemplateFiles.Filename) } &quot;" IgnoreStandardErrorWarningFormat="true"/>
</Target>
{% endhighlight %}

The Target order needed to be specified twice: a first time via BeforeTargets="ResolveAssemblyReferences" so the code gets generated before XAML compilation and a second time via CompileDependsOn to get Intellisense to work.

A couple more points of interest in that XML fragment:

- An ItemGroup sub-element inside Target is used to add the generated files to the list of C# files to compile.
- The target specifies both Inputs and Outputs attributes to enable incremental builds and avoid regenerating C# files when not needed. This is quite useful as PowerShell takes some time to start. The downside is that MSBuild does not detect registry-key modifications, so a forced rebuild is required for those changes to be applied.

Besides sluggishness, one other issue with using PowerShell inside MSBuild is that the latter does not understand well the error messages from the former. That tends to create obscure error messages in build logs. Using an [MSBuild task](http://www.codeproject.com/Articles/42853/Strongly-typed-AppSettings-with-MSBuild) instead of a script would likely improve that. This is left as an exercise for the reader.

PowerShell script
---

The second part of the code is the PowerShell script. It takes a pair of XML files, merge them, merge environment variables, and generate the C# class definition. The main stumbling block there was the use of [ref] variables, needed for some reason to avoid PowerShell duplicating XML elements when calling the Update-Settings() function.

{% highlight powershell %}
param([String]$inputPath, [String]$outputPath, [String]$namespace, [String]$class)

function Update-Settings([ref]$settingsRef, $overrideSettings)
{
    $settings = $settingsRef.value
    $key = $settings.key
    
    # Replace value with environment variable if present
    if (Test-Path env:$key)
    {
        $settings.SetAttribute("value", (Get-Item env:$key).Value)
    }
    
    # Replace value with override value if present
    if ($overrideSettings -ne $null)
    {
        if (($overrideSettings.Count -eq $null) -and ($overrideSettings.key -eq $key))
        {
            $settings.SetAttribute("value", $overrideSettings.value)
        }
        else
        {
            $index = $overrideSettings.key.IndexOf($key)
            if ($index -ge 0)
            {
                $settings.SetAttribute("value", $overrideSettings[$index].value)
            }
        }
    }
    
    # Verify value present
    if ($settings.value -eq $null)
    {
        throw "Could not find a value for setting $key"
    }    
}

# Load XML settings template
$template = [xml](Get-Content $inputPath)
$settings = $template.settings.set

# Load optional XML settings override
$overrideSettings = $null
if (($template.settings.override -ne $null) -and (Test-Path $template.settings.override))
{
    $override = [xml](Get-Content $template.settings.override)
    $overrideSettings = $override.settings.set
}

# Apply override values and environment variables
if ($settings.Count -eq $null)
{
    Update-Settings ([ref]$settings) $overrideSettings
}
else
{
    for ($i = 0; $i -lt $settings.Count; $i++)
    {
        Update-Settings ([ref]$settings[$i]) $overrideSettings
    }
}

# Generate C# settings file
'using System;' | Out-File $outputPath
'' | Out-File $outputPath -Append
('namespace ' + $namespace) | Out-File $outputPath -Append
'{' | Out-File $outputPath -Append
('    internal static class ' + $class) | Out-File $outputPath -Append
'    {' | Out-File $outputPath -Append
if ($settings.Count -eq $null)
{
    ('        public const String ' + $settings.key + ' = "' + $settings.value + '";') | Out-File $outputPath -Append
}
else
{
    for ($i = 0; $i -lt $settings.Count; $i++)
    {
        ('        public const String ' + $settings[$i].key + ' = "' + $settings[$i].value + '";') | Out-File $outputPath -Append
    }
}
'    }' | Out-File $outputPath -Append
'}' | Out-File $outputPath -Append
{% endhighlight %}
