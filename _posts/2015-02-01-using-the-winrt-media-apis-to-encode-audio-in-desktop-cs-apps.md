---
layout: post
title: Using the WinRT Media APIs to encode audio in Desktop C# apps
comments: true
---

The Windows Runtime (WinRT) provides the [MediaStreamSource](https://msdn.microsoft.com/en-us/library/windows/apps/windows.media.core.mediastreamsource.aspx) and [MediaTranscoder](https://msdn.microsoft.com/en-us/library/windows/apps/windows.media.transcoding.mediatranscoder.aspx) classes to encode audio/video streams in Universal Phone/Windows Store apps. Those classes are not limited to Store apps though: they can also be used in Desktop apps (command line, WPF, etc.) via a few tricks.

# Enabling WinRT APIs in Desktop C# apps

The first trick is to enable Windows Runtime support in Desktop C# projects. Toward that end, open the .csproj file in your favorite text editor and:

- in the first `<PropertyGroup>` add:

{% highlight XML %}
<TargetPlatformVersion>8.1</TargetPlatformVersion>
{% endhighlight %}

- in the same `<PropertyGroup>` also bump the framework version to 4.5.1:

{% highlight XML %}
<TargetFrameworkVersion>v4.5.1</TargetFrameworkVersion>
{% endhighlight %}

- in the `<ItemGroup>` containing `<Reference>` elements add:

{% highlight XML %}
<Reference Include="Windows" />
<Reference Include="System.Runtime" />
<Reference Include="System.Runtime.InteropServices.WindowsRuntime" />
<Reference Include="System.Runtime.WindowsRuntime" />
{% endhighlight %}

Then save, go to Visual Studio, and reload the project (Visual Studio automatically prompts for that).

# Accessing known folders

The second trick is to work around WinRT APIs which are tied to Store apps. For instance most of [StorageFile](https://msdn.microsoft.com/en-us/library/windows/apps/windows.storage.storagefile.aspx) and [StorageFolder](https://msdn.microsoft.com/en-us/library/windows/apps/windows.storage.storagefolder.aspx) work in Desktop apps but [KnownFolders](https://msdn.microsoft.com/en-us/library/windows/apps/windows.storage.knownfolders.aspx) has issues. The work around is to get paths to known folders using [Environment.GetFolderPath()](https://msdn.microsoft.com/en-us/library/system.environment.getfolderpath(v=vs.110).aspx) first and then pass those paths to [StorageFolder.GetFolderFromPathAsync()](https://msdn.microsoft.com/en-us/library/windows/apps/windows.storage.storagefolder.getfolderfrompathasync.aspx) to open a `StorageFolder`:

{% highlight csharp %}
StorageFolder musicFolder = await StorageFolder.GetFolderFromPathAsync(
    Environment.GetFolderPath(Environment.SpecialFolder.MyMusic)
    );
{% endhighlight %}

# Encoding audio data

Besides that, the code to encode audio is the same in Desktop apps as in Store apps. First create a `MediaStreamSource` specifying the input audio format via [AudioEncodingProperties](https://msdn.microsoft.com/en-us/library/windows/apps/windows.media.mediaproperties.audioencodingproperties.aspx) and generating audio data in the [SampleRequested](https://msdn.microsoft.com/en-us/library/windows/apps/windows.media.core.mediastreamsource.samplerequested.aspx) event handler:

{% highlight csharp %}
var source = new MediaStreamSource(
    new AudioStreamDescriptor(
        generator.EncodingProperties
        )
    );

source.SampleRequested += (MediaStreamSource sender, MediaStreamSourceSampleRequestedEventArgs args) =>
{
    args.Request.Sample = generator.GenerateSample();
};
{% endhighlight %}

Then create a `MediaTranscoder`, passing the media source, a destination stream, and a destination encoding profile (say M4A):

{% highlight csharp %}
var transcoder = new MediaTranscoder();
var result = await transcoder.PrepareMediaStreamSourceTranscodeAsync(
    source,
    destination,
    MediaEncodingProfile.CreateM4a(AudioEncodingQuality.Medium)
    );
await result.TranscodeAsync();
{% endhighlight %}

# Source code

For more details, see the full code sample [here](
https://github.com/mmaitre314/MediaReader/blob/master/MediaCaptureReader/UnitTestsCs.Desktop/MediaStreamSourceTests.cs).
