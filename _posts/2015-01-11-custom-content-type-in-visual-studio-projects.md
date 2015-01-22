---
layout: post
title: "Custom Content Type in C++ Visual Studio Projects"
comments: true
---

For the [TraceEtw](https://github.com/mmaitre314/TraceEtw) project I needed to generate code (headers, manifests, etc.) during the build and have the build consume those files on the fly. It turns out that Visual Studio C++ projects (.vcxproj) and MSBuild support a high level of customization for both the build process and the IDE UI. Moreover, this customization is all contained in the project files so it can be neatly packaged using NuGet to make it reusable. This blog brings together the various pieces needed to make that work.

Customization begins by defining a new file extension (say .epx) and associate it with a content type in an XML config file (here called EventProvider.xml):

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<ProjectSchemaDefinitions xmlns="http://schemas.microsoft.com/build/2009/properties">

  <ContentType Name="EventProvider" DisplayName="Event Provider" ItemType="EventProvider" />
  <ItemType Name="EventProvider" DisplayName="Event Provider" />
  <FileExtension Name=".epx" ContentType="EventProvider" />
  
</ProjectSchemaDefinitions>
{% endhighlight %}

This piece of XML associates the .epx file extension with the `<EventProvider>` item element in MSBuild project files.

The XML file is referenced either in a .targets file (for later reuse and NuGet packaging) or in the VS project itself (for one-time use):

{% highlight xml %} 
<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

  <ItemGroup>
    <PropertyPageSchema Include="$(MSBuildThisFileDirectory)EventProvider.xml" />
    <AvailableItemName Include="EventProvider">
      <Targets>TraceEtwGenerate</Targets>
    </AvailableItemName>
  </ItemGroup>

</Project>
{% endhighlight %}

For NuGet packaging the .targets file needs to have the same name as the package ID so that NuGet adds it to the build project during package install. In NuGet's .nuspec config file, the .targets file is placed in the build folder:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<package xmlns="http://schemas.microsoft.com/packaging/2011/08/nuspec.xsd">
  <metadata>
    <id>MMaitre.TraceEtw</id>
    ...
  </metadata>
  <files>
    <file src="MMaitre.TraceEtw.targets" target="build\native\" />
    <file src="EventProvider.xml" target="build\native\" />
    ...
  </files>
</package>
{% endhighlight %}

At that point, when an .epx file gets added to the project it is assigned its own `<EventProvider>` item element instead of some the usual `<Xml>`, `<None>`, `<ClCompile>`, etc.:

![Visual Studio project]({{ site.url }}/images/2015-01-11-VsProject.png)

{% highlight xml %}    
<ItemGroup>
    <EventProvider Include="EtwLogger.epx"/>
</ItemGroup>
{% endhighlight %}

This allows referencing those files as `@(EventProvider)` lists in MSBuild targets and customizing their processing.

Visual Studio also allows the definition of custom UI property pages to give more control over the build to users without having them edit the build project XML:
   
![Property Page]({{ site.url }}/images/2015-01-11-PropertyPage.png)

This is achieved by adding a `<Rule>` element under `<ProjectSchemaDefinitions>` in the first XML file mentioned in this blog:
    
{% highlight xml %}
<Rule Name="EventProvider" DisplayName="Event Provider" Order="500" PageTemplate="tool"   >
    <Rule.Categories>
        <Category Name="General" DisplayName="General" />
    </Rule.Categories>
    <Rule.DataSource>
        <DataSource Persistence="ProjectFile" ItemType="EventProvider" HasConfigurationCondition="true" />
    </Rule.DataSource>
    <BoolProperty Name="Verbose" DisplayName="Verbose" Description="Specifies verbose output." Category="General" Default="false" />
</Rule>
{% endhighlight %}
    
This piece of XML defines a boolean property called 'Verbose' that appears in the 'General' property page of files whose content type is 'EventProvider'. When the user modifies that property its new value gets attached to the existing `<EventProvider>` element in the build project:

{% highlight xml %}
<ItemGroup>
    <EventProvider Include="EtwLogger.epx">
        <Verbose Condition="'$(Configuration)|$(Platform)'=='Release|Win32'">true</Verbose>
    </EventProvider>
</ItemGroup>
{% endhighlight %}

The property value is retrieved using `%(EventProvider.Verbose)` in MSBuild targets:
  
{% highlight xml %}
<Target Name="TraceEtwGenerate"
      Inputs="@(EventProvider)"
      Outputs="@(EventProvider -> '$(ProjectDir)Events\%(Filename).man')" >
    <Message Importance="high" Text="Processing %(EventProvider.Filename).epx" />
    <GenerateEventProvider  InputXmlPath="%(EventProvider.FullPath)"
                            Verbose="%(EventProvider.Verbose)"
                            />
</Target>
{% endhighlight %}
   
For full code samples, see the following files on GitHub:

- [EventProvider.xml](https://github.com/mmaitre314/TraceEtw/blob/master/TraceEtw/EventProviderGenerator/EventProvider.xml) - definition of content type and property page
- [MMaitre.TraceEtw.targets](https://github.com/mmaitre314/TraceEtw/blob/master/TraceEtw/EventProviderGenerator/MMaitre.TraceEtw.targets) - inclusion in build project
- [MMaitre.TraceEtw.nuspec](https://github.com/mmaitre314/TraceEtw/blob/master/TraceEtw/EventProviderGenerator/MMaitre.TraceEtw.nuspec) - NuGet packaging
